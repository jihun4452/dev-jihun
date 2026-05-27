## Zookeeper와 Controller Broker

### Zookeeper란

분산 시스템 간의 정보를 신속하게 공유하기 위한 코디네이션 시스템이다. 클러스터 내 개별 노드의 중요한 상태 정보를 관리하고, 분산 시스템에서 리더 노드를 선출하는 역할을 수행한다. 개별 노드 간 상태 정보의 동기화를 위한 복잡한 Lock 관리 기능도 제공한다. Zookeeper 자체의 클러스터링 기능도 있다.

### Z Node

Zookeeper는 간편한 디렉토리 구조 기반의 Z Node를 활용한다. Z Node는 개별 노드의 중요 정보를 담고 있다.

개별 노드들은 Zookeeper의 Z Node를 계속 모니터링한다. Z Node에 변경이 발생하면 Watch Event가 트리거되어 변경 정보가 개별 노드들에게 통보된다.

### Kafka 클러스터에서 Zookeeper의 역할

세 가지 핵심 역할을 담당한다.

- **Controller Broker 선출(Election)**: Controller는 여러 브로커들에서 파티션 Leader 선출을 수행한다.
- **Broker Membership 관리**: 클러스터의 브로커 List, 브로커 Join/Leave 관리 및 통보를 담당한다.
- **Topic 정보 관리**: 토픽의 파티션, replicas 등의 정보를 가진다.

모든 카프카 브로커는 주기적으로 Zookeeper에 접속하면서 Session Heartbeat을 전송하여 자신의 상태를 보고한다. Zookeeper는 `zookeeper.session.timeout.ms` 이내에 Heartbeat을 받지 못하면 해당 브로커의 노드 정보를 삭제하고 Controller 노드에게 변경 사실을 통보한다.

### Controller Broker

Zookeeper에 가장 먼저 접속을 요청한 브로커가 Controller가 된다. Controller는 파티션에 대한 Leader Election을 수행하는 역할을 맡는다. Zookeeper로부터 브로커 추가/Down 등의 정보를 받으면, 해당 브로커로 인해 영향을 받는 파티션들에 대해 새로운 Leader Election을 수행한다.

Controller 자신이 Down되면 모든 노드에게 해당 사실이 통보되고, 가장 먼저 Zookeeper에 접속한 브로커가 새 Controller가 된다.

### Leader Election 수행 프로세스

Broker #3가 Shutdown되는 상황을 예로 들면 아래 순서로 진행된다.

1. Broker #3가 Shutdown된다. Zookeeper는 session 기간 동안 Heartbeat이 오지 않으므로 해당 브로커 노드 정보를 갱신한다.
2. Controller는 Zookeeper를 모니터링하던 중 Watch Event로 Broker #3의 Down 정보를 받는다.
3. Controller는 다운된 브로커가 관리하던 파티션들에 대해 새로운 Leader/Follower를 결정한다. 예를 들어 Partition #3의 Leader를 Broker #2로 변경한다.
4. 결정된 새로운 Leader/Follower 정보를 Zookeeper에 저장한다.
5. 해당 파티션을 복제하는 모든 브로커들에게 새로운 Leader/Follower 정보를 전달하고, 새로운 Leader로부터 복제를 수행할 것을 요청한다.
6. 모든 브로커가 가지는 MetadataCache를 새로운 Leader/Follower 정보로 갱신할 것을 요청한다.
