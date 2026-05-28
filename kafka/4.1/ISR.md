## ISR(In-Sync Replicas)과 min.insync.replicas

### ISR이란

Follower들은 누구라도 Leader가 될 수 있다. 단, ISR 내에 있는 Follower들만 가능하다.

파티션의 Leader 브로커는 Follower 파티션 브로커들이 Leader가 될 수 있는지 지속적으로 모니터링하여 ISR을 관리한다. Leader의 메시지를 빠르게 복제하지 못하고 뒤처지는 Follower는 ISR에서 제거되고, 차기 Leader 후보에서도 빠진다.

### ISR 조건

두 가지 조건을 모두 만족해야 ISR에 속할 수 있다.

- **Zookeeper 연결 유지**: `zookeeper.session.timeout.ms`로 지정된 기간(기본 6초, 최대 18초) 내에 Heartbeat을 지속적으로 Zookeeper로 보내야 한다.
- **복제 속도 유지**: `replica.lag.time.max.ms`로 지정된 기간(기본 10초, 최대 30초) 내에 Leader의 메시지를 지속적으로 가져가야 한다.

### Fetch 요청 방식

Follower는 Leader에게 Fetch 요청을 수행한다. 이 요청에는 Follower가 다음에 읽을 메시지의 offset 번호가 포함된다. Leader는 Follower가 요청한 offset 번호와 현재 Leader 파티션의 가장 최신 offset 번호를 비교한다. 이를 통해 Follower가 얼마나 복제를 잘 수행하고 있는지 판단한다.

예를 들어 Leader의 최신 offset이 999인데 Follower가 offset 999를 요청했다면 잘 따라오고 있는 거다. 반면 계속 뒤처진 offset을 요청한다면 ISR에서 제거 대상이 된다.

ISR은 파티션별로 독립적으로 관리된다. Follower가 복제를 따라가지 못하면 해당 Follower만 ISR에서 빠진다. 예를 들어 Partition #1의 ISR이 `Br #1, Br #2, Br #3`이었는데 Br #3이 느려지면 `Br #1, Br #2`만 남는 식이다.

### min.insync.replicas

`min.insync.replicas`는 브로커 설정값이다. Producer가 `acks=all`로 성공적으로 메시지를 보낼 수 있는 최소한의 ISR 브로커 개수를 의미한다.

`replication-factor=3`, `min.insync.replicas=2`로 설정된 경우를 예로 든다.

브로커 1개가 다운되어 ISR이 2개로 줄어도 조건을 만족하므로 정상 전송된다. 브로커 2개가 다운되어 ISR이 1개만 남으면 `min.insync.replicas=2` 조건을 충족하지 못한다. 이때 Producer는 `NOT_ENOUGH_REPLICAS` 에러를 받는다.

일반적으로 아래 조합을 권장한다.

- `replication-factor=3`
- `min.insync.replicas=2`
- `acks=all`

이렇게 하면 브로커 1개가 다운돼도 정상 동작하고, 2개가 동시에 다운되면 에러를 발생시켜서 메시지 손실을 방지할 수 있다.
