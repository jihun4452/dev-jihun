## 카프카는 오래된 메시지를 어떻게 정리하는가 — Log Cleanup Policy

앞 글에서 메시지가 Segment 단위로 저장된다는 걸 봤다. 그런데 메시지를 계속 받기만 하면 디스크는 언젠가 가득 찬다. 카프카는 오래된 Segment를 정리하는 정책을 `log.cleanup.policy`로 제공한다. (Topic 레벨에서는 `cleanup.policy`)

### 두 가지 정리 방식 — delete vs compact

```
log.cleanup.policy = delete            → 시간/크기 기준으로 Segment 통째 삭제
log.cleanup.policy = compact           → key별 최신 메시지만 남기고 재구성
log.cleanup.policy = [delete, compact] → 둘을 함께 적용
```

- **delete**: `log.retention.hours`나 `log.retention.bytes` 설정값에 따라 오래된 Segment를 그냥 삭제한다. 가장 일반적인 방식이다.
- **compact**: 같은 key를 가진 메시지 중 가장 최신 것만 남기도록 Segment를 재구성한다. (compaction의 상세 동작은 별도 글에서 다뤘다.)

이 글에서는 기본값이자 가장 많이 쓰는 **delete** 정책을 중심으로 본다.

### delete 정책의 핵심 설정

| 설정 | 설명 |
|------|------|
| `log.retention.hours` (ms) | Segment가 삭제되기 전 유지되는 시간. 기본 1주일(168시간). Topic Config는 `retention.ms` |
| `log.retention.bytes` | 삭제 조건이 되는 **파티션 단위** 전체 파일 크기. 기본값 `-1`(무한대). Topic Config는 `retention.bytes` |
| `log.retention.check.interval.ms` | 브로커가 백그라운드로 삭제 대상 Segment를 찾는 주기 |

여기서 헷갈리기 쉬운 두 가지를 짚자.

**첫째, 시간과 크기는 OR 조건이다.** `log.retention.hours`와 `log.retention.bytes` 중 어느 하나라도 조건에 걸리면 삭제 대상이 된다. 시간을 길게 잡아도 크기 제한에 먼저 걸리면 삭제될 수 있다.

**둘째, `log.retention.bytes`는 파티션 단위다.** 토픽 전체가 아니라 개별 파티션 기준이라는 점을 놓치면 디스크 사용량 계산이 어긋난다. 파티션이 많은 토픽이라면 실제 사용량은 `log.retention.bytes × 파티션 수`에 가까워진다.

### 트레이드오프 — 얼마나 오래 들고 있을 것인가

retention 설정은 결국 **디스크 비용**과 **데이터 보존 기간** 사이의 줄다리기다.

- 크게 설정하면: 오래된 Segment를 계속 들고 있으니 과거 데이터 재처리(consumer가 과거 offset부터 다시 읽기)에 유리하지만 디스크를 많이 먹는다.
- 작게 설정하면: 디스크는 아끼지만, retention 기간을 넘긴 메시지는 더 이상 조회할 수 없다.

특히 consumer가 한동안 죽어 있다가 살아났을 때, 읽으려던 offset의 Segment가 이미 retention으로 삭제됐다면 그 데이터는 영영 못 읽는다. 그래서 retention은 "최악의 경우 consumer가 얼마나 밀릴 수 있는가"를 감안해서 잡아야 한다.

### Active Segment는 건드리지 않는다

한 가지 더. Cleanup 정책은 **closed된 Segment에만 적용된다.** 현재 write가 일어나는 Active Segment는 retention 시간이 지나도 삭제 대상이 되지 않는다. 그래서 메시지 유입이 거의 없는 토픽에서는, retention을 짧게 잡아도 Active Segment가 롤링되기 전까지는 데이터가 남아 있는 것처럼 보일 수 있다. 이건 버그가 아니라 설계된 동작이다.

이 delete 정책과 짝을 이루는 게 compact 정책이고, 둘의 차이를 알면 "변경 이력이 중요한 토픽"과 "최신 상태만 중요한 토픽"을 어떻게 다르게 관리할지 판단할 수 있게 된다.
