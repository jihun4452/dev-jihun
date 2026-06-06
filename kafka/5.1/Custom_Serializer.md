## 카프카로 객체를 주고받기 — Custom Serializer/Deserializer 구현

카프카 브로커는 메시지를 **바이트 배열(byte array)** 로만 취급한다. Producer가 보내는 것도, 토픽 파티션에 저장되는 것도, Consumer가 읽는 것도 전부 바이트 배열이다. 그래서 객체를 보내려면 "객체 → 바이트 배열"(직렬화)과 "바이트 배열 → 객체"(역직렬화)를 거쳐야 한다.

```
[Producer]  객체 → Serializer → byte[]  →→ 브로커(byte[]) →→  byte[] → Deserializer → 객체  [Consumer]
```

### 기본 제공 Serializer로는 부족하다

카프카는 Primitive 타입에 대한 Serializer/Deserializer를 기본 제공한다.

- `StringSerializer`
- `ShortSerializer`
- `IntegerSerializer`
- `LongSerializer`
- `DoubleSerializer`
- `BytesSerializer`

문제는 실무 데이터가 이렇게 단순하지 않다는 것이다. 주문 정보, 고객 정보 같은 업무 객체(`OrderModel`, `Customer` 등)를 통째로 주고받으려면 **직접 Custom Serializer/Deserializer를 구현**해야 한다.

예를 들어 이런 주문 모델을 보낸다고 하자.

```java
public class OrderModel {
    public String orderId;
    public String shopId;
    public String menuName;
    public String userName;
    public String phoneNumber;
    public String address;
    public LocalDateTime orderTime;
    // 생성자, getter/setter
}
```

이걸 그대로 보낼 수는 없다. `OrderModel` → `byte[]` 변환 로직이 필요하다.

### 구현은 3단계

1. **Custom 객체 정의** — 비어있는 기본 생성자와 getter/setter를 갖춘 모델 클래스
2. **Serializer / Deserializer 구현** — `Serializer<T>`, `Deserializer<T>` 인터페이스를 구현
3. **Kafka Client에 등록** — Producer/Consumer Properties에 구현한 클래스를 지정

### 방법 1 — Pure Java로 직접 바이트를 다루기

가장 원초적인 방법은 멤버 변수를 하나하나 바이트로 변환해서 `ByteBuffer`에 채우는 것이다.

```java
public byte[] serialize(String topic, Order data) {
    if (data == null) return null;

    byte[] serializedName = data.getName().getBytes(encoding);
    int sizeOfName = serializedName.length;
    byte[] serializedDate = data.getStartDate().toString().getBytes(encoding);
    int sizeOfDate = serializedDate.length;

    ByteBuffer buf = ByteBuffer.allocate(4 + 4 + sizeOfName + 4 + sizeOfDate);
    buf.putInt(data.getID());
    buf.putInt(sizeOfName);
    buf.put(serializedName);
    buf.putInt(sizeOfDate);
    buf.put(serializedDate);
    return buf.array();
}
```

동작은 하지만 단점이 명확하다. 필드가 하나 추가될 때마다 직렬화/역직렬화 코드를 둘 다 고쳐야 하고, "각 필드의 길이를 먼저 쓰고 그다음 값을 쓴다" 같은 규약을 양쪽이 정확히 맞춰야 한다. 필드가 많아지면 실수하기 딱 좋다.

### 방법 2 — Jackson Databind로 JSON 직렬화 (권장)

실무에서는 보통 **Jackson Databind**를 쓴다. Java 객체 ↔ JSON 변환 라이브러리인데, 객체를 JSON 바이트로 한 줄에 직렬화할 수 있다.

```java
ObjectMapper objectMapper = new ObjectMapper();

// Serializer
public byte[] serialize(String topic, OrderModel order) {
    byte[] serializedOrder = null;
    try {
        serializedOrder = objectMapper.writeValueAsBytes(order);
    } catch (JsonProcessingException e) {
        logger.error("Json processing exception: " + e.getMessage());
    }
    return serializedOrder;
}
```

역직렬화는 그 반대다. 바이트 배열을 받아서 다시 객체로 복원한다.

```java
// Deserializer
public OrderModel deserialize(String topic, byte[] data) {
    if (data == null) return null;
    OrderModel order = null;
    try {
        order = objectMapper.readValue(data, OrderModel.class);
    } catch (IOException e) {
        logger.error("Json deserialize exception: " + e.getMessage());
    }
    return order;
}
```

Pure Java 방식과 비교하면, 필드가 늘어도 모델 클래스만 고치면 되고 직렬화 코드는 건드릴 필요가 없다. 가독성과 유지보수성에서 비교가 안 된다.

### 직렬화와 역직렬화는 한 쌍이다

여기서 꼭 기억할 점. **Serializer와 Deserializer는 항상 짝으로 맞아야 한다.** Producer가 Jackson으로 JSON 직렬화했는데 Consumer가 Pure Java 방식으로 역직렬화하려 하면 당연히 깨진다. 같은 규약(같은 모델 클래스, 같은 직렬화 방식)을 양쪽이 공유해야 한다.

그래서 보통 모델 클래스와 Serializer/Deserializer를 **공통 모듈로 빼서** Producer 프로젝트와 Consumer 프로젝트가 함께 의존하도록 구성한다. 이렇게 해두면 모델이 바뀌어도 양쪽이 자동으로 같은 정의를 쓰게 된다.

### 흔한 함정 — LocalDateTime 직렬화 오류

Jackson을 쓸 때 자주 만나는 에러가 있다. `OrderModel`의 `orderTime` 같은 `java.time.LocalDateTime` 필드를 직렬화하려 하면 이런 메시지가 뜬다.

```
Java 8 date/time type `java.time.LocalDateTime` not supported by default:
add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310" to enable handling...
```

기본 Jackson은 Java 8의 날짜/시간 타입(`java.time.*`)을 직렬화하지 못한다. 해결책은 에러 메시지 그대로다. **`jackson-datatype-jsr310` 모듈을 추가**하고 `ObjectMapper`에 등록해주면 된다.

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.registerModule(new JavaTimeModule());
```

`LocalDateTime`, `LocalDate`, `Instant` 등을 쓴다면 이 모듈 등록은 거의 필수라고 보면 된다. 처음 Custom 객체에 시간 필드를 넣었을 때 한 번쯤 꼭 마주치는 함정이니 미리 알아두면 시간을 아낄 수 있다.
