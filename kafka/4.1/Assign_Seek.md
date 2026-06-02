## 특정 파티션 할당과 특정 Offset부터 읽기

### 특정 파티션만 할당하기

컨슈머에게 여러 개의 파티션이 있는 토픽에서 특정 파티션만 할당할 수 있다. 배치 처리 시 특정 key 레벨의 파티션을 특정 컨슈머에 할당하여 처리할 경우에 사용한다.

`KafkaConsumer`의 `assign()` 메소드에 `TopicPartition` 객체로 특정 파티션을 인자로 입력하여 할당한다.

```java
TopicPartition topicPartition = new TopicPartition(topicName, 0);
kafkaConsumer.assign(Arrays.asList(topicPartition));
```

### 특정 Offset부터 읽기

특정 메시지가 누락되었을 경우 해당 메시지를 다시 읽어오기 위해 유지보수 차원에서 일반적으로 사용한다. `TopicPartition` 객체로 할당할 파티션을 설정하고 `seek()` 메소드로 읽어올 offset을 지정한다.

```java
TopicPartition topicPartition = new TopicPartition(topicName, 1);
kafkaConsumer.assign(Arrays.asList(topicPartition));
kafkaConsumer.seek(topicPartition, 6L);
```

### 유의사항

기존 group id와 동일한 group id를 사용하면서 커밋을 수행하면 `__consumer_offsets`을 재갱신하게 된다. seek으로 특정 offset부터 다시 읽어서 처리한 뒤 커밋까지 수행하면 기존 offset 정보가 덮어써지므로 주의해야 한다.
