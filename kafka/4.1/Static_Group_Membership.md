## Static Group Membership과 Rebalancing Protocol

### Rebalance가 문제가 되는 이유

Consumer Group에 Consumer가 많을수록 Rebalance 시 모든 Consumer가 참여하므로 시간이 오래 걸린다. 대량 데이터 처리 중 Rebalance가 발생하면 그 시간만큼 메시지를 읽지 못하고 Lag가 쌓인다. 유지보수 차원의 단순 Consumer 재기동도 Rebalance를 유발하기 때문에 불필요한 Rebalance를 방지할 방법이 필요하다.

### Rebalance가 발생하는 상황

- Consumer Group 내에 새로운 Consumer가 추가되거나 기존 Consumer가 종료될 때
- Topic에 새로운 Partition이 추가될 때
- `session.timeout.ms` 이내에 Heartbeat 응답이 없을 때
- `max.poll.interval.ms` 이내에 `poll()` 메소드가 호출되지 않을 때

### Static Group Membership

Consumer Group 내의 Consumer들에게 고정된 id를 부여하는 방식이다. `group.instance.id`를 설정하면 된다.

최초 조인 시 할당된 파티션을 그대로 유지하고, Consumer가 shutdown되어도 `session.timeout.ms` 내에 재기동되면 Rebalance 없이 기존 파티션이 그대로 재할당된다.

Consumer #3가 종료되어도 Rebalance가 일어나지 않으며 Partition #3는 다른 Consumer에 재할당되지 않고 잠시 읽히지 않는 상태가 된다. Consumer #3가 `session.timeout.ms` 내에 다시 기동되면 Partition #3는 Consumer #3에 그대로 할당된다. `session.timeout.ms` 내에 기동되지 않으면 그때 Rebalance가 수행되고 Partition #3가 다른 Consumer에 할당된다.

Static Group Membership을 적용할 경우 `session.timeout.ms`를 좀 더 큰 값으로 설정하는 게 좋다. 재기동 시간을 충분히 확보해야 불필요한 Rebalance를 막을 수 있기 때문이다.

### Eager Protocol

Rebalance 수행 시 기존 Consumer들의 모든 파티션 할당을 일제히 취소하고 잠시 메시지를 읽지 않는다. 이후 새롭게 파티션을 다시 할당받고 메시지를 읽는다.

모든 Consumer가 동시에 멈추는 구간이 생기므로 Lag가 크게 발생할 가능성이 있다. Range, Round Robin, Sticky 할당 전략이 여기에 해당한다.

### Cooperative Protocol

Rebalance 수행 시 모든 파티션 할당을 취소하지 않는다. 재할당이 필요한 파티션만 대상으로 점진적으로 Consumer를 할당하면서 Rebalance를 수행한다.

전체 Consumer가 메시지 읽기를 중지하지 않으며 영향을 받는 파티션만 재분배된다. Consumer가 많은 Group에서 Rebalance 시간이 길어질 때 활용도가 높다. Cooperative Sticky 할당 전략이 여기에 해당한다.

두 프로토콜의 핵심 차이는 Rebalance 중 나머지 Consumer들이 계속 메시지를 처리할 수 있느냐다. Eager는 전부 멈추고, Cooperative는 영향받는 것만 멈춘다.
