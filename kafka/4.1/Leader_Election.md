## Preferred Leader Election과 Unclean Leader Election

### Preferred Leader Election

파티션별로 최초에 할당된 Leader/Follower 브로커 설정을 Preferred Broker라고 한다. 브로커가 shutdown 후 재기동될 때 이 Preferred Leader Broker를 일정 시간 이후에 다시 선출한다.

`auto.leader.rebalance.enable=true`로 설정하고 `leader.imbalance.check.interval.seconds`를 일정 시간으로 설정하면 된다. 기본값은 300초다.

예를 들어 Partition #1의 Preferred Leader가 Broker #1이라면, Broker #1이 재기동된 이후 300초가 지나면 자동으로 Leader를 Broker #1로 복원한다. Preferred Leader 자체는 변하지 않는다.

### 문제 상황

Broker #2, #3가 모두 shutdown되면 Partition #1, #2, #3의 Leader가 Broker #1로 몰린다. 이 상태에서 Broker #1에 메시지가 계속 유입된다. 그런데 Broker #1까지 shutdown되면 문제가 생긴다.

이후 Broker #2, #3이 재기동되어도 Partition #1, #2, #3의 Leader가 될 수 없다. ISR에서 제외된 상태이기 때문이다. Broker #1이 가진 메시지를 갖고 있지 않아서다.

```
Broker #1 (Down)    Broker #2 (재기동)    Broker #3 (재기동)
Offset 1000         Offset 800            Offset 800
                    → ISR 아님, Leader 불가
```

### Unclean Leader Election

기존 Leader 브로커가 오랫동안 살아나지 않는 경우, Out of Sync 상태인 Follower 브로커를 Leader로 선출할지 결정해야 한다.

`unclean.leader.election.enable=true`로 설정하면 복제가 완료되지 않은 Follower도 Leader가 될 수 있다. 대신 기존 Leader가 가지고 있던 메시지 손실이 발생한다.

- `true`: 가용성 우선. 메시지 손실을 감수하고 서비스를 유지한다.
- `false`: 일관성 우선. Leader가 살아날 때까지 해당 파티션은 동작하지 않는다.

어떤 걸 선택할지는 서비스의 성격에 따라 다르다. 금융처럼 데이터 손실이 절대 안 되는 경우라면 `false`가 맞다. 반면 로그 수집처럼 일부 유실이 허용되는 경우라면 `true`로 해서 가용성을 유지하는 게 나을 수 있다.
