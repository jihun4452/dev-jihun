## Consumer는 한 번의 poll에서 얼마나 가져올까 — Fetcher 파라미터 파헤치기

`poll()`을 호출하면 메시지가 온다. 그런데 "한 번에 얼마나" 가져오는지는 사실 여러 파라미터의 줄다리기로 결정된다. 처리량이 안 나오거나, 반대로 한 번에 너무 많이 가져와서 `max.poll.interval.ms`에 걸리는 문제는 대부분 이 Fetcher 파라미터를 모르고 기본값으로 둬서 생긴다. 이번엔 Fetcher가 데이터를 가져오는 메커니즘과 관련 파라미터를 정리한다.

### Fetcher와 ConsumerNetworkClient의 분업

먼저 구조부터. Consumer가 `poll()`을 호출하면 동작은 이렇게 나뉜다.

- **Fetcher**: 내부 Linked Queue에서 데이터를 꺼내 반환한다. Queue에 데이터가 없으면 ConsumerNetworkClient에게 "브로커에서 가져와달라"고 요청한다.
- **ConsumerNetworkClient**: 비동기 I/O로 브로커에서 메시지를 가져와 Linked Queue에 채워 넣는다.

즉 Fetcher는 직접 브로커를 때리지 않는다. Queue를 보고, 비어 있으면 요청만 건다. 실제 네트워크 I/O는 ConsumerNetworkClient가 비동기로 돌린다. 이 분리 덕분에 `poll()`이 매번 네트워크를 기다리며 멈추지 않고, 미리 채워둔 Queue에서 바로 꺼내 줄 수 있다.

### "얼마나, 얼마나 기다려" — 가져오는 양과 대기 시간

브로커에서 데이터를 가져올 때의 양과 타이밍은 다음 파라미터들이 정한다.

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `fetch.min.bytes` | 1 | Fetcher가 읽어들이는 최소 bytes. 브로커는 이 크기 이상 쌓일 때까지 전송을 미룬다 |
| `fetch.max.wait.ms` | 500 | `fetch.min.bytes`만큼 쌓이길 기다리는 최대 시간 |
| `fetch.max.bytes` | 50MB | Fetcher가 한 번에 가져올 수 있는 최대 데이터 bytes |
| `max.partition.fetch.bytes` | 1MB | 파티션별로 한 번에 가져올 수 있는 최대 bytes |
| `max.poll.records` | 500 | 한 번의 `poll()`에서 가져올 수 있는 최대 레코드 수 |

`fetch.min.bytes`와 `fetch.max.wait.ms`는 한 쌍이다. "최소 이만큼 모이면 보내라, 근데 안 모여도 이 시간 지나면 그냥 보내라"는 의미다. `fetch.min.bytes`를 키우면 브로커가 데이터를 모아서 한 번에 보내므로 처리량(throughput)은 올라가지만, 모일 때까지 기다리니 지연(latency)은 늘어난다. 전형적인 throughput vs latency 트레이드오프다.

### 상황별로 어떻게 동작하나

파라미터들이 맞물려 도는 모습을 시나리오로 보면 이해가 쉽다.

- **가져올 데이터가 1건도 없으면**: `poll()`에 넘긴 인자 시간만큼 대기했다가 빈 채로 return한다.
- **최신 offset을 따라가는 중이라면(실시간 처리)**: `fetch.min.bytes`만큼 모이면 보내고, 안 모이면 `fetch.max.wait.ms`까지 기다린 뒤 return한다.
- **오래된 과거 offset을 대량으로 읽는 중이라면**: 파티션에서 최대 `max.partition.fetch.bytes`만큼 읽어서 반환한다. 단 그만큼 못 채워도 최신 offset에 도달하면 거기서 반환한다.
- **파티션이 아무리 많아도**: 전체적으로 가져오는 양은 `fetch.max.bytes`로 제한된다.
- **Linked Queue에서 꺼내는 레코드 개수는**: `max.poll.records`로 제한된다.

### 튜닝 포인트

여기서 실무 감각 하나. `max.poll.records`는 처리 루프와 직결된다. `poll()`로 500건을 가져왔는데 한 건 처리에 시간이 오래 걸리는 작업(DB 저장 등)이 끼면, 500건 다 처리하기 전에 다음 `poll()`을 못 부르고 `max.poll.interval.ms`(기본 5분)를 넘겨 Rebalance가 터질 수 있다.

이럴 땐 두 가지 선택지가 있다.

- `max.poll.records`를 줄여서 한 번에 가져오는 양을 낮춘다. → 루프가 빨리 돌아 `poll()` 주기를 지킨다.
- `max.poll.interval.ms`를 늘린다. → 한 배치 처리에 시간을 더 준다.

처리 로직이 무거울수록 `max.poll.records`를 낮추는 쪽이 안전하다. 반대로 가볍고 빠른 처리라면 `fetch.min.bytes`를 키워 throughput을 끌어올리는 게 이득이다. 결국 Fetcher 파라미터 튜닝은 "내 Consumer가 한 건을 처리하는 데 얼마나 걸리는가"에서 출발한다.
