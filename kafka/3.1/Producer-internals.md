## Kafka Producer 배치 전송 튜닝 정리

### Sender 스레드와 배치

Sender 스레드는 1개만 있어도 바로 가져가고, 다 차지 않아도 가져간다. 여러 개의 batch를 가져갈 수도 있고, 안 찼는데 바로 가져갈 수도 있다.

그래서 "배치가 준비됐다고 바로 가져가지 말고, 승객 좀 쌓이면 가는 게 어때?"라는 것이다. `linger.ms`와 `batch.size`로 보통 사이즈 조절을 한다.

### linger.ms

`linger.ms`를 사용해서 "조금 채워서 갑시다"라고 정의한다. 일정 시간 대기해서 Record Batch에 메시지를 보다 많이 채울 수 있도록 적용한다.

`linger.ms`를 반드시 0보다 크게 설정할 필요는 없다. 프로듀서와 브로커 간의 전송이 매우 빠르고, 프로듀서에서 메시지가 적절히 Record Accumulator에 누적된다면 0이어도 무방하다. 프로듀서와 브로커 간의 전송이 느리다면 `linger.ms`를 높여서 메시지가 배치로 적용될 수 있는 확률을 높이는 시도를 해볼 만하다. `linger.ms`는 보통 20ms 이하로 설정하기를 권장한다.

`linger.ms`가 0이면 안 기다리고 버퍼가 차 있으면 가져간다는 의미다.

### 관련 설정들

Record Accumulator가 send할 때 배치를 정리해서 넣어준다. 바로 가는 것이 아니고 `max.request.size`만큼 정해서 가져갈 수도 있다. 한번 보낼 때 보내는 메시지의 최대 크기가 `max.request.size`이다.

Sender 스레드의 경우 `max.in.flight.requests.per.connection`을 2로 하면 최대 2개의 배치를 가져와라 라는 의미다. 5개면 5개의 배치. `buffer.memory`가 Record Accumulator의 전체 사이즈다.