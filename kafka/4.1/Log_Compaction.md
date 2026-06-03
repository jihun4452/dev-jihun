## Log Compaction

### 개념

`log.cleanup.policy=compact`로 설정하면 세그먼트의 key값에 따라 가장 최신 메시지로만 compact하게 세그먼트를 재구성한다. key값이 null인 메시지에는 적용할 수 없다. 백그라운드 스레드 방식으로 별도의 I/O 작업을 수행하므로 추가적인 I/O 부하가 소모된다.

### 수행 구조

파티션은 Log Compaction이 이미 적용된 클린(clean) 영역과 아직 적용되지 않은 더티(dirty) 영역으로 나뉜다. Log Cleaner가 dirty 영역을 대상으로 Compaction 작업을 수행한다. `log.cleaner.enabled=true`로 설정해야 동작한다.

Active Segment는 Compaction 대상에서 제외된다. Compaction은 파티션 레벨에서 수행 여부가 결정되며, 개별 세그먼트들을 새로운 세그먼트들로 재생성하는 방식으로 동작한다.

예를 들어 Segment #1(offset 0~999), #2(offset 1000~1999), #3(offset 2000~2999)이 있다면 Compaction 후 offset 0~2999 범위의 하나의 세그먼트로 재생성된다.

### Compaction 수행 후

- 메시지의 순서(Ordering)는 여전히 유지된다.
- 메시지의 offset은 변하지 않는다.
- Consumer는 Active Segment를 제외하고 메시지의 가장 최신값을 읽는다.
- 단, key값은 있지만 value가 null인 메시지(tombstone)는 일정 시간 이후에 삭제되기 때문에 읽지 못할 수 있다.

### Compaction 수행 조건

두 가지 조건 중 하나를 만족하면 Compaction이 수행된다.

- dirty 비율이 `log.cleaner.min.cleanable.ratio` 이상이고, 메시지가 생성된 지 `log.cleaner.min.compaction.lag.ms`가 지난 dirty 메시지에 수행된다.
- 메시지가 생성된 지 `log.cleaner.max.compaction.lag.ms`가 지난 dirty 메시지에 수행된다.

### 관련 설정

| 설정 | 설명 |
|------|------|
| `log.cleaner.min.cleanable.ratio` | Compaction을 수행하기 위한 dirty 데이터 비율 (dirty/total). 기본값 0.5. 값이 작을수록 더 빈번히 수행. Topic Config는 `min.cleanable.dirty.ratio` |
| `log.cleaner.min.compaction.lag.ms` | 메시지 생성 후 최소 이 시간이 지나야 Compaction 대상이 됨. 기본값 0. Topic Config는 `min.compaction.lag.ms` |
| `log.cleaner.max.compaction.lag.ms` | dirty ratio 이하여도 이 시간이 지나면 Compaction 대상이 됨. 기본값은 사실상 무한대. Topic Config는 `max.compaction.lag.ms` |

### 메시지 삭제 처리

key값은 있지만 value가 null인 메시지를 tombstone이라고 한다. Compaction 수행 시 해당 메시지는 내부적으로 tombstone으로 표시된다. 이후 `log.cleaner.delete.ms`가 지나면 해당 메시지가 완전히 삭제된다.

예를 들어 offset 37에 `Key="Min", Value="AAA"`, offset 39에 `Key="Min", Value="BBB"`, offset 40에 `Key="Min", Value=null`이 있다면, Compaction 후 offset 40의 tombstone만 남고 나머지는 제거된다. 이후 `log.cleaner.delete.ms`가 지나면 offset 40도 삭제된다.
