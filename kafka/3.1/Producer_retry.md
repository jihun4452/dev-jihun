## Producer 메시지 전송/재 전송 시간 파라미터 이해

### 전체 관계

`delivery.timeout.ms` ≥ `linger.ms` + `request.timeout.ms`

Sender 스레드가 메시지 전송하다가 오류나면 재전송하고, 응답 없으면 재전송하는데, 언제까지 하느냐. `delivery.timeout.ms`만큼만 하고 더 이상 재전송하지 않는다. 이 값이 전체 전송 과정의 최종 마감 시간이다. 아무리 재시도를 해도 이 시간을 넘기면 포기하고 Exception을 던진다.

### Record Accumulator에 넣는 단계

send를 호출하면 Record Accumulator에 넣으면 이제 끝이고, 나머지는 Sender 스레드가 알아서 한다. 그런데 Record Accumulator가 꽉 차면 send를 못 집어넣는다. 이때 `max.block.ms`까지 비기를 기다린다. 그래도 못 집어넣으면 바로 Exception을 던진다.

`max.block.ms`는 Record Accumulator에 입력하지 못하고 블락되는 최대 시간이다. 메시지 생산 속도가 전송 속도보다 빨라서 버퍼가 계속 차 있으면 이 타임아웃에 걸릴 수 있다. 이 경우 프로듀서 쪽에서 전송 속도를 높이거나, `buffer.memory`를 늘리거나, 메시지 생산 속도를 조절해야 한다.

### Sender 스레드가 배치를 가져가는 단계

`linger.ms`는 Sender 스레드가 배치별로 가져가기 위한 최대 대기 시간이다. 이 시간만큼 기다려서 배치에 메시지를 좀 더 채운 뒤 가져간다. 0이면 안 기다리고 바로 가져간다.

### 브로커에 전송하고 응답을 기다리는 단계

`request.timeout.ms`는 브로커에 전송한 뒤 응답을 기다리는 최대 시간이다. 전송 재시도 대기 시간은 제외하고, 이 시간을 초과하면 리트라이 하거나 타임아웃 Exception이 발생한다.

`retry.backoff.ms`는 전송 실패 후 다음 재시도까지 쉬는 대기 시간이다. 바로 재시도하면 브로커에 부담을 줄 수 있으니까, 이만큼 기다렸다가 다시 시도한다.

### 재전송 예시

retry=10이고, `retry.backoff.ms`=30ms, `request.timeout.ms`=10000ms일 때를 보면, 한 번 전송에 최대 10000ms 기다리고, 실패하면 30ms 쉬고 다시 보낸다. 이걸 최대 10회 반복한다. 그런데 10회를 다 채우기 전이라도 `delivery.timeout.ms`에 도달하면 더 이상 시도하지 않는다. 결국 `delivery.timeout.ms`가 모든 재시도를 포함한 최종 마감 시간이다.

### 전체

send() 호출 → `max.block.ms`만큼 버퍼 대기 → Record Accumulator에 저장 → `linger.ms`만큼 배치 대기 → Sender 스레드가 가져감 → 브로커에 전송 → `request.timeout.ms`만큼 응답 대기 → 실패 시 `retry.backoff.ms`만큼 쉬고 재시도 → 전체 과정이 `delivery.timeout.ms`를 넘기면 포기