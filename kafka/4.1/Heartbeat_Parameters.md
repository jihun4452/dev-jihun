## Heartbeat와 poll 관련 주요 파라미터

### Heartbeat Thread

카프카 컨슈머는 내부적으로 Heartbeat Thread를 별도로 운영한다. 메인 스레드와 분리되어 독립적으로 동작하며, 브로커의 Group Coordinator에게 Consumer의 상태를 주기적으로 전송하는 역할을 한다.

Heartbeat가 일정 시간 내에 도달하지 않으면 브로커는 해당 Consumer에 문제가 있다고 판단하고 Rebalance를 지시한다.

### 주요 파라미터

**heartbeat.interval.ms** (기본값: 3000ms)

Heartbeat Thread가 Heartbeat를 보내는 간격이다. `session.timeout.ms`보다 낮게 설정되어야 하며, `session.timeout.ms`의 1/3보다 낮게 설정하는 것을 권장한다. 너무 짧으면 네트워크 부하가 늘고, 너무 길면 브로커가 Consumer 상태를 늦게 파악한다.

**session.timeout.ms** (기본값: 45000ms)

브로커가 Consumer로부터 Heartbeat를 기다리는 최대 시간이다. 이 시간 동안 Heartbeat를 받지 못하면 브로커는 해당 Consumer를 Group에서 제외하도록 Rebalance 명령을 지시한다. Static Group Membership을 사용하는 경우 재기동 시간을 고려해서 이 값을 충분히 크게 설정해야 한다.

**max.poll.interval.ms** (기본값: 300000ms)

이전 `poll()` 호출 후 다음 `poll()` 호출까지 브로커가 기다리는 최대 시간이다. 이 시간 동안 `poll()` 호출이 오지 않으면 브로커는 해당 Consumer에 문제가 있다고 판단하고 Rebalance 명령을 보낸다.

`poll()`로 가져온 데이터를 DB에 저장하는 등 시간이 걸리는 작업을 메인 스레드에서 수행할 경우, 그 처리 시간이 `max.poll.interval.ms`를 초과하면 의도치 않은 Rebalance가 발생한다. 처리 시간이 긴 작업이 있다면 이 값을 넉넉하게 설정하거나 처리 로직을 별도 스레드로 분리하는 것을 고려해야 한다.

### 파라미터 간 관계

세 파라미터는 서로 맞물려 동작한다. `heartbeat.interval.ms < session.timeout.ms` 관계를 반드시 유지해야 하고, `session.timeout.ms`의 1/3 이하로 `heartbeat.interval.ms`를 설정하는 게 일반적인 권장 사항이다. `max.poll.interval.ms`는 메시지 처리 로직의 최대 소요 시간을 감안해서 설정한다.
