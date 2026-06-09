## Consumer가 처음 붙으면 어디서부터 읽을까 — auto.offset.reset과 offset 보존

Consumer를 새로 띄웠을 때 "처음부터 읽을지, 지금부터 읽을지"를 정하는 게 `auto.offset.reset`이다. 단순해 보이지만 `__consumer_offsets`에 저장된 offset 정보와 맞물리면서 의외로 헷갈리는 동작들이 있다. 직접 데여보기 전에 정리해두면 좋은 주제다.

### earliest vs latest

`auto.offset.reset`은 Consumer가 읽을 offset 정보가 없을 때, 파티션의 어디서부터 읽을지를 정한다.

```
auto.offset.reset = earliest  →  파티션의 가장 처음 offset부터 읽음
auto.offset.reset = latest    →  파티션의 가장 마지막(최신) offset 이후부터 읽음
```

`earliest`는 "쌓여 있던 과거 데이터까지 다 읽겠다", `latest`는 "지금 이 순간 이후 들어오는 것만 읽겠다"는 뜻이다.

참고로 CLI에서 `kafka-console-consumer`를 쓸 때는 `--from-beginning` 옵션을 줘야만 `earliest`로 동작한다. 안 주면 latest다.

### 중요한 전제 — "offset 정보가 없을 때"만 적용된다

여기가 핵심이다. `auto.offset.reset`은 **`__consumer_offsets`에 해당 Consumer Group의 offset 정보가 없을 때만** 작동한다.

무슨 말이냐면, 동일한 `group.id`로 Consumer가 다시 접속하면, 카프카는 `__consumer_offsets`에 저장된 그 그룹의 마지막 offset부터 읽어온다. 이때는 `auto.offset.reset`을 `earliest`로 설정해놨더라도 **0번 offset부터 읽지 않는다.** 이미 "어디까지 읽었다"는 기록이 있으니 그 뒤부터 이어 읽는 게 맞기 때문이다.

`auto.offset.reset`이 실제로 힘을 쓰는 경우는 이런 상황이다.

- 완전히 새로운 `group.id`로 처음 접속할 때
- 기존 offset 정보가 만료되어 사라졌을 때

"`earliest`로 했는데 왜 처음부터 안 읽지?" 하는 혼란은 거의 다 이 전제를 놓쳐서 생긴다. 이미 커밋된 offset이 있으면 `auto.offset.reset`은 무시된다.

### offset은 영원히 보관되지 않는다 — offsets.retention.minutes

그럼 offset 정보는 얼마나 보관될까. Consumer Group의 모든 Consumer가 종료되어도, 그 그룹이 읽었던 offset 정보는 바로 사라지지 않고 `__consumer_offsets`에 **일정 기간 저장**된다. 이 기간이 `offsets.retention.minutes`이고, 기본값은 **7일**이다.

이게 실무에서 함정이 되는 지점이 있다. 어떤 Consumer Group을 일주일 넘게 안 돌리다가 다시 띄우면, 그 사이 offset 정보가 만료되어 사라졌을 수 있다. 그러면 `auto.offset.reset` 설정이 다시 적용되면서, `earliest`라면 처음부터, `latest`라면 최신부터 읽게 된다. 의도와 다르게 과거 데이터를 통째로 다시 읽거나, 반대로 쌓여 있던 데이터를 건너뛰는 사고가 여기서 난다.

### 토픽을 지웠다 다시 만들면

한 가지 더. 해당 토픽이 삭제되고 재생성되면, 그 토픽에 대한 Consumer Group의 offset 정보는 `__consumer_offsets`에 **0으로 기록**된다. 토픽이 물리적으로 새로 태어났으니 기존 offset이 의미가 없어진 것이다. 이 경우에도 결국 `auto.offset.reset` 기준으로 다시 시작점을 잡게 된다.

### 정리

`auto.offset.reset`은 "기본 시작점"이 아니라 **"기댈 offset 정보가 없을 때의 비상 규칙"** 이라고 이해하는 게 정확하다. 평소엔 `__consumer_offsets`의 커밋된 offset이 우선이고, 그게 없거나(새 그룹) 사라졌을 때(retention 만료, 토픽 재생성) 비로소 `auto.offset.reset`이 무대에 오른다. 이 우선순위만 머릿속에 박아두면 Consumer 시작 지점 때문에 당황할 일이 없다.
