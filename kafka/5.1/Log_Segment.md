## 카프카 로그는 어떻게 디스크에 저장될까 — Segment와 Index

카프카를 쓰다 보면 "토픽에 메시지를 보낸다"는 말을 자주 한다. 그런데 그 메시지가 실제로 디스크에 어떤 모습으로 떨어지는지는 잘 들여다보지 않는다. 이번 글에서는 파티션 디렉토리 안으로 한번 들어가 본다.

### 파티션은 디렉토리, 메시지는 Segment

흔히 "메시지가 파티션에 저장된다"고 말하지만, 엄밀히 따지면 파티션은 그냥 **파일 디렉토리**일 뿐이다. 실제 메시지는 그 디렉토리 안의 **Segment 파일**에 저장된다.

하나의 파티션은 여러 개의 Segment로 구성된다. Segment는 데이터 용량이 일정 크기를 넘거나 일정 시간이 지나면 close되고, 새로운 Segment가 생성되어 그다음 메시지를 이어서 받는다.

여기서 중요한 규칙이 하나 있다. **하나의 파티션은 단 하나의 Active Segment만 가진다.** 브로커는 이 Active Segment에만 write를 수행하고, 한번 close된 Segment는 read-only가 되어 더 이상 기록되지 않는다.

```
Partition #1
 ├─ Segment #1   (closed, read-only)
 ├─ Segment #2   (closed, read-only)
 └─ Segment #3   (active, write/read)   ← 새 메시지는 여기로만
```

### Segment를 닫는 기준 — log.segment.bytes와 log.roll.hours

그럼 Segment는 언제 닫히고 새로 만들어질까. 두 가지 설정이 이를 결정한다.

| 설정 | 설명 |
|------|------|
| `log.segment.bytes` | 개별 Segment의 최대 크기. 기본값 1GB. 이 크기를 넘기면 close되고 새 Segment가 생성된다. Topic Config는 `segment.bytes` |
| `log.roll.hours` (ms) | 개별 Segment가 유지되는 최대 시간. 기본값 7일. 크기가 다 차지 않았더라도 이 시간이 지나면 close된다. Topic Config는 `segment.ms` |

즉 **"크기가 다 찼거나, 시간이 다 됐거나"** 둘 중 하나만 만족해도 Segment는 롤링(rolling)되어 닫힌다. 메시지 유입이 적은 토픽이라도 `log.roll.hours` 덕분에 무한정 하나의 Segment에만 쌓이지는 않는다.

### Segment 파일명의 비밀

Segment 파일명은 해당 Segment가 담고 있는 **첫 메시지의 offset**을 기반으로 만들어진다. 20자리 0 패딩 형식이다.

```
00000000000000000000.log   →  offset 0 ~ 41
00000000000000000042.log   →  offset 42 ~ 83
00000000000000000084.log   →  offset 84 ~ ?   (active)
```

파일명만 봐도 이 Segment가 어느 offset부터 시작하는지 바로 알 수 있다. 덕분에 특정 offset을 찾을 때 어떤 Segment 파일을 열어야 할지 빠르게 결정할 수 있다.

### .index와 .timeindex — offset을 빠르게 찾기 위한 장치

토픽을 생성하면 파티션 디렉토리에는 `.log` 파일만 생기는 게 아니다. 세 종류의 파일이 함께 만들어진다.

- **`.log`** — 실제 메시지 내용을 담는 Segment 파일
- **`.index`** — offset별 byte position 정보
- **`.timeindex`** — 메시지 생성 시간(Unix time, ms)별 offset 정보

`.index`가 왜 필요할까. Segment는 결국 파일이다. 특정 offset의 메시지를 읽으려면 "파일의 시작 지점에서 몇 byte 떨어진 위치에 그 메시지가 있는지"를 알아야 한다. 이 byte position을 매번 처음부터 스캔해서 찾으면 너무 느리다. 그래서 `.index`가 offset → byte position 매핑을 미리 들고 있는 것이다.

다만 `.index`는 **모든 offset에 대한 정보를 갖지는 않는다.** `log.index.interval.bytes`(기본 4096)에 설정된 byte만큼 Segment가 쌓일 때마다 한 번씩 해당 offset의 위치를 기록한다. 즉 듬성듬성 기록해두고, 그 사이는 순차 스캔으로 메우는 방식이다. 인덱스를 촘촘하게 만들면 조회는 빨라지지만 인덱스 파일 자체가 커지므로, 이 둘 사이의 트레이드오프를 잡은 설정이다.

```
.index                          .timeindex
offset  →  byte position        timestamp        →  offset
  18    →    4334                1661158661643    →    18
  36    →    8615                1661158698705    →    36
```

`.timeindex`는 "특정 시간 이후의 메시지부터 읽고 싶다"는 요구를 처리하기 위한 것이다. 타임스탬프 기반 조회나 Log Retention 처리에서 활용된다.

### Segment의 생명주기 — Active → Closed → Deleted / Compacted

정리하면 Segment 파일은 다음 단계를 거친다.

```
Active  →  Closed  →  Deleted (or Compacted)
```

1. **Active**: 현재 write가 일어나는 Segment. 파티션당 하나뿐이다.
2. **Closed**: 크기/시간 조건으로 롤링되어 닫힌 Segment. read-only.
3. **Deleted / Compacted**: Log Cleanup 정책(`log.cleanup.policy`)에 따라, 일정 시간/크기가 지나면 삭제되거나 key 기준 최신값만 남기는 Compaction 대상이 된다.

이 마지막 단계인 Cleanup 정책(delete / compact)은 다음 글에서 자세히 다룬다. 핵심은 **카프카가 메시지를 영원히 쌓아두지 않고, Segment 단위로 잘라서 관리·삭제한다**는 점이다. 이 Segment 구조를 이해하고 나면 Retention 설정이나 Compaction 동작이 훨씬 자연스럽게 읽힌다.
