## 카프카 설정은 어디서 관리되는가 — Config 체계와 kafka-configs

카프카를 다루다 보면 설정값이 정말 많다. `acks`, `linger.ms`, `retention.ms`, `min.insync.replicas`... 그런데 이 설정들이 다 같은 곳에서 관리되는 게 아니다. 어디서 설정하고 어떻게 바꾸는지를 구분해서 이해하면, "이 값을 바꾸려면 재기동을 해야 하나?" 같은 질문에 바로 답할 수 있다.

### 크게 두 갈래 — 브로커/토픽 레벨 vs 클라이언트 레벨

| 구분 | 설명 |
|------|------|
| **Broker / Topic 레벨 Config** | 카프카 **서버**에서 설정되는 Config. Topic Config는 Broker Config를 기본값으로 따르되, Topic별로 따로 지정하면 그 값을 우선한다 |
| **Producer / Consumer 레벨 Config** | 카프카 **클라이언트**에서 설정되는 Config. `server.properties`에 존재하지 않으며, 클라이언트 실행 시마다 코드/Properties로 지정한다 |

핵심 구분선은 **"이 설정이 서버 쪽 것인가, 클라이언트 쪽 것인가"** 이다.

- `log.retention.hours`, `min.insync.replicas`, `num.partitions` → 서버(Broker/Topic) 레벨
- `acks`, `linger.ms`, `batch.size`, `group.id`, `auto.offset.reset` → 클라이언트(Producer/Consumer) 레벨

클라이언트 레벨 Config는 `kafka-configs`로 수정할 수 없다. 어차피 클라이언트가 실행될 때 코드에서 지정하기 때문이다.

### Static Config vs Dynamic Config

서버 레벨 Config 안에서도 또 나뉜다.

- **Static Config**: `server.properties` 파일에 있는 설정. 변경하려면 **브로커 재기동이 필요**하다.
- **Dynamic Config**: `kafka-configs` 명령어로 **동적으로 변경 가능**한 설정. 재기동 없이 적용된다.

운영 중인 클러스터를 재기동하는 건 부담이 크다. 그래서 자주 조정할 만한 값(예: 특정 토픽의 retention)은 Dynamic Config로 바꾸는 게 안전하다. 어떤 설정이 Dynamic으로 변경 가능한지는 카프카 버전별 문서에서 확인할 수 있다.

### kafka-configs 사용법

`kafka-configs` 명령어로 Config를 조회/변경/삭제할 수 있다.

**Config 값 확인**

```bash
kafka-configs --bootstrap-server [host:port] \
  --entity-type [brokers/topics] \
  --entity-name [broker id / topic name] \
  --all --describe
```

**Config 값 설정**

```bash
kafka-configs --bootstrap-server [host:port] \
  --entity-type [brokers/topics] \
  --entity-name [broker id / topic name] \
  --alter --add-config property명=value
```

**Config 값 해제(Unset)**

```bash
kafka-configs --bootstrap-server [host:port] \
  --entity-type [brokers/topics] \
  --entity-name [broker id / topic name] \
  --alter --delete-config property명
```

`--entity-type`으로 brokers인지 topics인지 지정하고, `--entity-name`으로 대상(브로커 id 또는 토픽명)을 지정하는 구조다.

### 우선순위를 기억하자

마지막으로 한 가지. Topic Config는 Broker Config를 **기본값으로 상속**하되, Topic에 명시적으로 값을 주면 그게 우선한다. 예를 들어 브로커 전체의 `log.retention.hours`가 168시간(1주)이라도, 특정 토픽에 `retention.ms`를 따로 걸면 그 토픽만 다른 보존 기간을 갖는다.

이 상속 구조 덕분에 "클러스터 전체의 기본 정책은 브로커에서 잡고, 예외가 필요한 토픽만 개별 설정으로 덮어쓴다"는 운영 패턴이 가능해진다. 설정이 의도대로 안 먹는 것 같을 때는, 토픽 레벨에서 덮어쓴 값이 있는지부터 `--describe`로 확인해보는 게 좋다.
