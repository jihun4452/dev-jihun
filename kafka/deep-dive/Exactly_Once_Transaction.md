## 진짜 "정확히 한 번"은 Transaction이 필요하다

idempotence 글을 정리하면서 마지막에 이런 한계를 적었다. "Consumer → Process → Producer 로직에서, Consumer가 offset을 커밋하지 못했는데 Producer는 메시지를 보냈다면 중복이 생길 수 있고, 이건 Transaction 기반 처리가 필요하다." 그 후속편이다. idempotence만으로는 왜 부족한지, Transaction이 어떻게 그 구멍을 메우는지 정리한다.

### idempotence의 한계 복습

idempotence(중복 없이 전송)는 강력하지만 **딱 한 가지 상황, Producer와 Broker 사이의 retry 시 중복만** 제거한다. 그 바깥은 책임지지 않는다.

- 같은 메시지를 `send()`로 두 번 호출하면? → 중복 제거 대상 아니다. 그냥 두 번 저장된다.
- Producer가 재기동되면? → Producer ID가 새로 바뀌므로, 이전에 보냈던 메시지를 또 보낼 수 있다.
- Consumer가 읽고 → 가공하고 → 다시 Produce하는 파이프라인에서, Consume한 offset 커밋과 Produce가 따로 논다면? → 한쪽만 성공하고 한쪽이 실패하면 중복이나 누락이 생긴다.

특히 마지막 **consume-transform-produce** 패턴이 문제다. "메시지를 읽어서, 가공해서, 다른 토픽으로 다시 보내는" 흐름인데, Kafka Streams 같은 게 대표적으로 이 모양이다. 여기서 진짜 "정확히 한 번(Exactly Once)"을 보장하려면 **읽은 offset 커밋과 보낸 메시지가 하나의 원자적 단위**로 묶여야 한다. 둘 다 성공하거나, 둘 다 없던 일이 되거나.

### Transaction이 묶어주는 것

Transaction은 여러 작업을 하나의 원자적(atomic) 단위로 처리한다. 카프카 트랜잭션이 묶는 건 두 가지다.

- 여러 파티션(여러 토픽 포함)에 대한 메시지 전송
- Consumer offset 커밋(`sendOffsetsToTransaction`)

이 둘을 한 트랜잭션으로 묶으면, "메시지 전송"과 "어디까지 읽었는지 기록"이 함께 commit되거나 함께 abort된다. 중간에 죽어서 한쪽만 반영되는 상황이 사라진다.

### 코드 흐름

Producer 쪽은 트랜잭션 API를 쓴다.

```java
// 설정: 트랜잭션을 쓰려면 고유한 transactional.id가 필요
props.setProperty(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-tx-id");
// enable.idempotence는 트랜잭션 사용 시 자동으로 true

producer.initTransactions();   // 최초 1회

try {
    producer.beginTransaction();

    // 가공한 메시지들 전송
    producer.send(record1);
    producer.send(record2);

    // 여기까지 읽은 Consumer offset도 같은 트랜잭션에 포함
    producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);

    producer.commitTransaction();   // 전송 + offset 커밋을 한꺼번에 확정
} catch (Exception e) {
    producer.abortTransaction();    // 문제 생기면 통째로 롤백
}
```

`commitTransaction()`이 호출돼야 비로소 전송과 offset 커밋이 함께 확정된다. 중간에 예외가 나면 `abortTransaction()`으로 전부 없던 일이 된다.

### 읽는 쪽도 약속을 지켜야 한다 — read_committed

Producer만 트랜잭션을 걸면 끝이 아니다. 그 결과를 읽는 Consumer도 "확정된 것만 읽겠다"고 선언해야 한다.

```java
props.setProperty(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

- `read_uncommitted` (기본값): 트랜잭션이 commit되지 않은 메시지도 그냥 읽는다.
- `read_committed`: commit된 트랜잭션의 메시지만 읽는다. abort된 메시지는 건너뛴다.

`read_committed`로 설정해야 abort된(롤백된) 메시지를 Consumer가 보지 않는다. 즉 Exactly Once는 **Producer의 트랜잭션 + Consumer의 read_committed**가 양쪽에서 맞물려야 완성된다.

### 내부에서 일어나는 일

내부적으로는 브로커에 **Transaction Coordinator**가 있고, 트랜잭션 상태는 `__transaction_state`라는 내부 토픽에 기록된다. (Consumer offset이 `__consumer_offsets`에 저장되는 것과 비슷한 구조다.)

그리고 트랜잭션이 commit/abort될 때 브로커는 해당 파티션에 **Control Message**(commit/abort 마커)를 기록한다. `read_committed` Consumer는 이 마커를 보고 "이 트랜잭션은 확정됐구나 / 롤백됐구나"를 판단해서 읽을지 말지를 결정한다.

### 공짜는 아니다

당연히 트랜잭션은 오버헤드가 있다. Coordinator와의 통신, control message 기록, `read_committed`로 인한 약간의 지연 등이 붙는다. 그래서 모든 곳에 트랜잭션을 거는 건 과하다.

정리하면 선택 기준은 이렇다.

- 단순 전송에서 retry 중복만 막으면 된다 → **idempotence**로 충분
- consume-transform-produce 파이프라인에서 offset과 전송을 원자적으로 묶어야 한다 → **Transaction** 필요

idempotence가 "한 구간(Producer↔Broker)의 중복 방지"라면, Transaction은 "읽기-가공-쓰기 전체를 한 묶음으로 보장"하는 것이다. 둘은 경쟁 관계가 아니라, 보장하려는 범위가 다른 도구다.
