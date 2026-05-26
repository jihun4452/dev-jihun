## 리플리케이션(Replication)

### 리플리케이션이란

카프카는 개별 노드의 장애를 대비하여 높은 가용성을 제공한다. 그 핵심이 리플리케이션(Replication), 즉 복제다.

Replication은 토픽 생성 시 `replication-factor` 설정값으로 구성한다. `replication-factor=3`이면 원본 파티션과 복제 파티션을 포함해서 총 3개의 파티션을 가진다는 의미다. `replication-factor`의 개수는 브로커 개수보다 클 수 없다.

Replication의 동작은 토픽 내 개별 파티션들을 대상으로 적용된다. 복제 대상인 파티션들은 1개의 Leader와 N개의 Follower로 구성된다.

### Leader와 Follower

Producer와 Consumer는 Leader 파티션을 통해서만 쓰기와 읽기를 수행한다. 파티션의 복제는 Leader에서 Follower 방향으로만 이뤄진다. Follower가 Leader의 파티션을 읽어가는 방식이다.

파티션 리더를 관리하는 브로커는 Producer/Consumer의 읽기/쓰기를 관리함과 동시에, Follower를 관리하는 브로커의 Replication도 함께 관리한다.

참고로 Kafka 2.4부터는 Consumer가 Follower에서도 읽어오는 follower fetching이 가능해졌다.

### 단일 파티션 복제

Producer가 `acks=all`로 메시지 A를 Leader에게 보낸다. Leader는 Follower들에게 복제를 수행하고, 완료되면 ACK를 Producer에게 돌려준다. 그 다음에 Producer가 메시지 B를 전송한다.

메시지 A가 모든 Replicator에 완벽하게 복사되었는지 확인한 후에 메시지 B를 전송하는 방식이다. 메시지 손실이 되지 않도록 모든 장애 상황을 감안한 전송 모드지만, ACK를 오래 기다려야 하므로 상대적으로 전송 속도가 느리다.

### 멀티 파티션 복제

파티션이 여러 개인 경우 각 파티션의 Leader는 서로 다른 브로커에 분산된다. 어떤 브로커는 특정 파티션의 Leader이면서 동시에 다른 파티션의 Follower가 된다.

예를 들어 파티션이 3개, 브로커가 3개인 경우 Broker #1은 Partition #1의 Leader, Broker #2는 Partition #2의 Leader, Broker #3은 Partition #3의 Leader가 된다. 나머지 파티션들은 각각 Follower로 가지게 된다.

이렇게 Leader를 분산시켜서 특정 브로커에 부하가 몰리는 걸 방지한다.
