## Offset Commit의 이해

### __consumer_offsets

`__consumer_offsets`에는 컨슈머 그룹이 특정 토픽의 파티션별로 읽기 커밋한 offset 정보를 가진다. 특정 파티션을 어느 컨슈머가 커밋했는지 정보는 가지지 않는다.

예를 들어 아래와 같이 저장된다.

- Consumer Group 1 / Topic A / Partition #0 → offset 6
- Consumer Group 1 / Topic A / Partition #1 → offset 8
- Consumer Group 1 / Topic A / Partition #2 → offset 10

### 중복 읽기 상황

Consumer #1이 poll로 메시지를 읽어들였지만 아직 커밋을 하지 않은 상태에서 Consumer #1이 종료되고 Rebalancing이 발생한다. Consumer #2는 마지막으로 커밋된 offset부터 다시 읽어온다. Consumer #1이 이미 읽어서 처리한 데이터를 Consumer #2가 다시 처리하게 된다. 중복 읽기가 발생한다.

### 읽기 누락 상황

Consumer #1이 poll로 읽어들이고 있는 도중 커밋을 먼저 수행한 경우, Consumer #1이 종료되면 Rebalancing 후 Consumer #2는 커밋된 offset 이후부터 읽어온다. Consumer #1이 아직 처리하지 못한 데이터는 누락된다.

### Auto Commit

`enable.auto.commit=true`인 경우 사용자가 명시적으로 커밋을 기술하지 않아도 컨슈머가 자동으로 커밋을 수행한다. `auto.commit.interval.ms`에 정해진 주기(기본 5초)마다 자동으로 커밋한다.

정확히는 auto.commit.interval.ms가 지난 후에 수행되는 poll에서 이전 poll에서 가져온 마지막 메시지의 offset을 커밋한다. 브로커의 커밋이 컨슈머가 읽어온 메시지보다 오래될 수 있으므로, 컨슈머 장애/재기동 및 Rebalancing 후 이미 읽어온 메시지를 다시 읽어와서 중복 처리될 수 있다.

### Manual Commit

`enable.auto.commit=false`로 설정하고 사용자가 직접 커밋을 제어한다. 동기 방식과 비동기 방식이 있다.

**Sync(동기 방식)**

`commitSync()` 메소드를 사용한다. poll로 메시지 배치를 읽어오고, 해당 메시지들의 마지막 offset을 브로커에 커밋 적용한다. 브로커에 커밋 적용이 성공적으로 될 때까지 블로킹된다. 커밋 완료 후에 다시 메시지를 읽어온다. 커밋 적용이 실패할 경우 다시 커밋 요청을 시도한다. 비동기 방식 대비 수행 시간이 느리다.

**Async(비동기 방식)**

`commitAsync()` 메소드를 사용한다. poll로 읽어오고 마지막 offset을 브로커에 커밋 요청하지만, 커밋 완료를 기다리지 않고 바로 다음 메시지를 읽어온다. 커밋이 실패해도 재시도하지 않는다. 따라서 컨슈머 장애 또는 Rebalance 시 한번 읽은 메시지를 다시 중복으로 가져올 수 있다. 동기 방식 대비 수행 시간이 빠르다.
