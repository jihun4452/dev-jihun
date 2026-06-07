## Group Coordinator는 Rebalance를 어떻게 지휘하는가

Consumer 글에서 "Rebalance가 일어난다"는 말을 계속 했는데, 정작 그 Rebalance를 누가 지휘하는지는 짚고 넘어가지 않았다. 그 주체가 바로 브로커 안에 있는 **Group Coordinator**다. 이번엔 Rebalance가 시작돼서 끝날 때까지 내부에서 무슨 일이 벌어지는지 정리한다.

### Group Coordinator의 역할

Group Coordinator는 브로커에 존재하며, Consumer Group을 관리하는 역할을 한다. 구체적으로 두 가지를 관리한다.

- Consumer들의 Join Group 정보와 파티션 매핑 정보
- Consumer들의 Heartbeat

Consumer Group 내에 새로운 Consumer가 추가되거나, 기존 Consumer가 종료되거나, Topic에 새로운 파티션이 추가되면 Group Coordinator가 Consumer들에게 파티션을 재할당하는 Rebalancing을 수행하도록 지시한다.

### Group Leader는 또 따로 있다

여기서 한 가지 짚고 갈 게 있다. **파티션을 실제로 할당하는 건 Group Coordinator가 아니다.** Coordinator는 지휘만 하고, 실제 할당 계산은 Consumer 중 하나인 **Group Leader**가 한다.

흐름을 순서대로 보면 이렇다.

1. Consumer Group 내의 Consumer가 브로커에 최초 접속을 요청하면 Group Coordinator가 생성된다.
2. 동일한 `group.id`로 여러 Consumer가 Group Coordinator에 접속한다.
3. 가장 먼저 Group에 Join 요청을 한 Consumer가 **Group Leader**로 지정된다.
4. Group Leader는 파티션 할당 전략(`partition.assignment.strategy`)에 따라 Consumer들에게 파티션을 할당한다.
5. Group Leader는 최종 할당 결과를 Group Coordinator에게 전달한다.
6. 전달이 성공적으로 공유되면, 개별 Consumer들은 자신에게 할당된 파티션에서 메시지를 읽기 시작한다.

즉 **Coordinator는 브로커 쪽, Leader는 Consumer 쪽**이다. 할당 정책을 브로커가 아니라 클라이언트(Consumer)가 결정하게 해둔 구조라, 새로운 할당 전략을 브로커 재배포 없이 클라이언트 라이브러리만으로 도입할 수 있다.

### Rebalance가 시작되는 트리거

Group Coordinator가 Rebalance를 지시하는 상황은 정해져 있다.

- Consumer Group에 새로운 Consumer가 Join할 때
- 기존 Consumer가 종료될 때
- Topic에 새로운 파티션이 추가될 때
- `session.timeout.ms` 내에 Heartbeat이 오지 않을 때
- `max.poll.interval.ms` 내에 `poll()`이 호출되지 않을 때

앞의 셋은 "구성이 바뀌었으니 다시 나눠야지"라는 정상적인 이유고, 뒤의 둘은 "이 Consumer 문제 있는 것 같은데?" 하고 쳐내는 경우다.

### Consumer Group의 상태 — empty, rebalance, stable

Group Coordinator는 각 Consumer Group의 상태(GroupMetaData)를 들고 있다. 상태는 크게 세 가지를 오간다.

```
empty  ──(새 Consumer Join)──▶  rebalance  ──(Rebalance 성공)──▶  stable
  ▲                                                                  │
  └──────────────(모든 Consumer가 떠남)◀──────────────────────────────┘
```

- **empty**: Consumer Group은 존재하지만 소속된 Consumer가 하나도 없는 상태.
- **rebalance**: 새 Consumer가 Join하거나, Heartbeat 문제·Consumer 종료·파티션 추가 등으로 재할당을 진행 중인 상태.
- **stable**: Rebalance가 끝나고 안정적으로 Consumer들이 메시지를 처리하는 상태.

운영하면서 Consumer가 잘 안 붙는 것 같을 때, Group이 계속 `rebalance` 상태에서 못 빠져나오는지(보통 Heartbeat이나 `max.poll.interval.ms` 문제) 확인해보면 원인이 보인다. 정상이라면 잠깐 `rebalance`를 거쳐 금방 `stable`로 떨어진다.

### 정리

결국 Rebalance는 **Coordinator(브로커)가 지시하고, Leader(Consumer)가 계산하고, 나머지 Consumer가 따르는** 협업 구조다. 이 역할 분담을 알고 나면 "왜 파티션 할당 전략이 클라이언트 설정인지", "왜 Heartbeat이 끊기면 Rebalance가 도는지"가 한 번에 이어진다.
