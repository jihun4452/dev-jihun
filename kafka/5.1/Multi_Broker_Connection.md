## Producer는 어떻게 여러 브로커에 접속하는가 — bootstrap.servers와 메타데이터

멀티 노드 클러스터를 구성하면 브로커가 여러 대가 된다. 그런데 Producer는 메시지를 보낼 때 "이 토픽 파티션의 Leader가 어느 브로커에 있는지" 알아야 한다. 파티션 Leader는 클러스터 상황에 따라 바뀌기도 하는데, Producer는 이걸 어떻게 따라갈까. 핵심은 `bootstrap.servers`와 **메타데이터**다.

### bootstrap.servers는 "시작점"일 뿐이다

Producer는 `bootstrap.servers`에 적힌 브로커 리스트를 기반으로 접속을 시작한다.

```java
props.setProperty(
    ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
    "192.168.56.101:9092, 192.168.56.101:9093, 192.168.56.101:9094"
);
```

이름이 `bootstrap`(부팅, 시작)인 게 핵심이다. 이 리스트는 **모든 메시지를 보낼 대상**이 아니라, **최초 접속해서 클러스터 정보를 얻어올 진입점**이다.

그래서 여기에 클러스터의 모든 브로커를 다 적을 필요는 없다. 몇 개만 적어도 된다. 다만 적어둔 브로커가 전부 죽어 있으면 진입 자체를 못 하므로, 보통 2~3개 이상을 적어서 한두 대가 죽어도 접속할 수 있게 해둔다.

### 브로커들은 메타데이터를 공유한다

개별 브로커들은 토픽 파티션의 **Leader와 Follower가 어디 있는지**에 대한 메타 정보를 서로 공유하고 있다.

```
Topic-A Partition 1 :  Leader = Br#1,  Follower = Br#2, Br#3
Topic-A Partition 2 :  Leader = Br#2,  Follower = Br#3, Br#1
Topic-A Partition 3 :  Leader = Br#3,  Follower = Br#1, Br#2
```

이 메타데이터는 어느 브로커에 물어봐도 같은 답을 준다. 그래서 Producer가 `bootstrap.servers`의 아무 브로커에나 처음 접속해도, 클러스터 전체의 파티션 배치 정보를 받아올 수 있다.

### 실제 접속 흐름

정리하면 이렇게 동작한다.

1. Producer가 `bootstrap.servers`에 적힌 브로커 중 하나에 최초 접속한다.
2. 그 브로커로부터 **메타데이터**(어떤 파티션의 Leader가 어느 브로커에 있는지)를 받아온다.
3. 보내려는 메시지의 토픽/파티션을 결정하고, 그 **파티션의 Leader가 있는 브로커로 다시 접속**해서 실제 메시지를 전송한다.

즉 `bootstrap.servers`에 적은 브로커와, 실제로 메시지를 보내는 브로커가 다를 수 있다. 진입은 아무 데로나 하지만, 실제 전송은 항상 해당 파티션의 Leader에게 간다. (Producer/Consumer의 읽기·쓰기는 Leader 파티션을 통해서만 이뤄진다는 복제 글의 내용과 이어진다.)

### Leader가 바뀌면?

브로커가 죽어서 파티션 Leader가 다른 브로커로 옮겨가는 상황(Leader Election)이 생기면, 기존 Leader 위치를 들고 있던 Producer의 메타데이터는 낡은 정보가 된다. 이때 Producer는 메타데이터를 갱신(refresh)해서 새 Leader 위치를 다시 받아오고, 그쪽으로 전송을 이어간다.

그래서 운영 중에 브로커 한 대가 내려가도, Producer는 메타데이터를 새로 받아 새 Leader로 붙기 때문에 (재시도 설정과 맞물려) 메시지 전송을 계속할 수 있다. `bootstrap.servers`에 여러 브로커를 적어두는 것과, 브로커들이 메타데이터를 공유하는 구조가 합쳐져서 이 가용성을 만들어낸다.
