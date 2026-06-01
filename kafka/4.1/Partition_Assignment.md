## Consumer 파티션 할당 전략

### 할당 전략 종류

카프카 컨슈머는 토픽의 파티션을 어떻게 나눠가질지 결정하는 할당 전략을 설정할 수 있다. 크게 네 가지가 있다.

- **Range**: 서로 다른 2개 이상의 토픽을 구독할 때 토픽별로 동일한 파티션을 특정 컨슈머에 할당하는 전략이다. 여러 토픽에서 동일한 키값으로 된 파티션들이 같은 컨슈머에 할당되어 해당 컨슈머가 여러 토픽의 동일 키값 데이터를 함께 처리할 수 있다.
- **Round Robin**: 여러 토픽의 파티션들을 컨슈머들에게 순차적으로 할당하는 전략이다. 컨슈머별로 균등한 부하 분배가 목적이다.
- **Sticky**: 최초 할당된 파티션-컨슈머 매핑을 Rebalance 이후에도 최대한 유지하는 전략이다. 단, Eager Protocol 기반이므로 Rebalance 시 모든 컨슈머의 파티션 매핑이 일제히 해제된 후 다시 매핑된다.
- **Cooperative Sticky**: Sticky와 유사하지만 Cooperative Protocol 기반이다. Rebalance 시 모든 매핑을 해제하지 않고 재할당이 필요한 파티션과 컨슈머만 재매핑한다.

### Round Robin vs Range 비교

**Case 1 — 파티션 수와 컨슈머 수가 동일한 경우**

Round Robin과 Range 모두 결과가 같다. 각 컨슈머가 토픽별로 동일한 파티션 번호를 가져간다.

- Consumer #1: Topic A P1, Topic B P1
- Consumer #2: Topic A P2, Topic B P2
- Consumer #3: Topic A P3, Topic B P3

**Case 2 — 파티션 수와 컨슈머 수가 다른 경우**

Round Robin은 파티션들을 순차적으로 돌아가며 균등 분배한다.

- Consumer #1: Topic A P1, P3 / Topic B P2
- Consumer #2: Topic A P2 / Topic B P1, P3

Range는 토픽별로 동일한 파티션 번호를 같은 컨슈머에 묶어서 할당한다.

- Consumer #1: Topic A P1, P2 / Topic B P1, P2
- Consumer #2: Topic A P3 / Topic B P3

Range는 컨슈머 간 불균등이 생길 수 있지만 동일 키값 데이터를 한 컨슈머에서 처리할 수 있는 이점이 있다.

### Rebalancing 시 동작 차이

**Round Robin**: Rebalance 후에도 균등 분배를 위해 기존 파티션-컨슈머 매핑이 바뀔 수 있다. Consumer #3가 빠지면 기존에 Consumer #1이 갖고 있던 파티션이 Consumer #2로 넘어가는 식이다.

**Sticky**: Rebalance 시 모든 매핑을 일제히 취소한 뒤, 기존 매핑은 다시 재매핑하고 Consumer #3가 갖고 있던 파티션만 재할당한다. 전체를 다 풀었다가 다시 붙이는 방식이라 비효율이 있다.

**Cooperative Sticky**: 모든 매핑을 취소하지 않는다. 재할당이 필요한 파티션만 순차적으로 Rebalance를 수행하여 할당한다. 기존 컨슈머들은 Rebalance 중에도 계속 메시지를 처리할 수 있다.
