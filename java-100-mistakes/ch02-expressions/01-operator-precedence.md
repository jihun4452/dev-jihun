## 연산자 우선순위와 괄호 (실수 001~003)

표현식 한 줄에서 가장 자주 터지는 실수가 연산자 우선순위 오해다. 컴파일은 멀쩡히 되는데 의도와 다르게 계산되는 게 문제다.

### 자바 연산자 우선순위

높은 것부터 낮은 순서로 정리하면 이렇다.

| 우선순위 | 연산자 |
|----------|--------|
| 높음 | 후위 `expr++` `expr--` |
| | 단항 `++expr` `--expr` `+` `-` `!` `~` |
| | 곱셈/나눗셈/나머지 `*` `/` `%` |
| | 덧셈/뺄셈 `+` `-` (문자열 결합 `+` 포함) |
| | 시프트 `<<` `>>` `>>>` |
| | 비교 `<` `>` `<=` `>=` `instanceof` |
| | 등치 `==` `!=` |
| | 비트 AND `&` |
| | 비트 XOR `^` |
| | 비트 OR `\|` |
| | 논리 AND `&&` |
| | 논리 OR `\|\|` |
| | 삼항 `? :` |
| 낮음 | 대입 `=` `+=` `-=` ... |

특히 헷갈리기 쉬운 지점들이다.

- **`+`/`-`가 시프트(`<<`)보다 높다.** `1 + 2 << 3`은 `(1 + 2) << 3`으로 해석된다.
- **비트 연산(`&` `^` `|`)이 등치(`==`)보다 낮다.** `flags & MASK == 0`은 `flags & (MASK == 0)`으로 해석되어 의도와 완전히 달라진다.
- **삼항 연산자(`? :`)는 거의 맨 아래다.** 그래서 `+`나 `==`와 섞이면 그쪽이 먼저 묶인다.

### 조건 표현식은 괄호로 묶어라

조건 표현식(삼항)이 더 큰 표현식의 일부로 들어갈 때는 **항상 괄호로 묶어야 한다.** 우선순위가 가장 낮은 축이라, 안 묶으면 옆의 연산자가 먼저 결합되기 때문이다.

### 조건 연산자와 null 검사

다음 코드를 보자.

```java
String format(String value) {
    return "Value:" + value != null ? value : "(unknown)";
}
```

원래 의도는 "`value`가 null이면 `(unknown)`, 아니면 `value`를 붙여서 반환"이다. 하지만 실제로는 이렇게 해석된다.

```java
return (("Value:" + value) != null) ? value : "(unknown)";
```

문자열 결합(`+`)이 `!=`나 삼항보다 우선순위가 높아서, `"Value:" + value`가 먼저 묶인다. 그런데 문자열 결합 결과는 **결코 null이 될 수 없으므로** 조건은 항상 참이고, 늘 `value`만 반환된다. (`value`가 null이면 그대로 null이 새어 나간다.)

**해결** — 의도한 부분을 괄호로 묶거나, 중간 변수로 분리한다.

```java
// 1) 괄호로 묶기
String format(String value) {
    return "Value:" + (value != null ? value : "(unknown)");
}

// 2) 중간 변수로 분리 (가독성 ↑)
String format(String value) {
    String display = value != null ? value : "(unknown)";
    return "Value:" + display;
}
```

### 문자열 결합이 먼저 묶이는 함정 — ArrayList 초기 용량

같은 우선순위 문제가 ElasticSearch 프로젝트에서도 발견된 적이 있다. `ArrayList`의 초기 용량을 계산하는 코드였는데, 비슷하게 재현하면 이렇다.

```java
List<String> trimAndAdd(List<String> input, String newItem) {
    List<String> result = new ArrayList<>(
        input.size() + newItem == null ? 0 : 1); // 잘못된 초기 용량 계산
    for (String s : input) {
        result.add(s.trim());
    }
    if (newItem != null) {
        result.add(newItem.trim());
    }
    return result;
}
```

의도는 "원소 개수만큼 미리 용량 확보"다. 그런데 `input.size() + newItem`이 먼저 묶인다. `int + String`이므로 **문자열 결합**이 되어 `"5null"` 같은 문자열이 만들어지고, 그 문자열은 null이 아니므로 삼항은 항상 `1`을 반환한다. 결국 용량이 늘 1로 잡힌다.

**해결** — 의도한 덧셈을 괄호로 묶고, 가독성을 위해 중간 변수로 뺀다.

```java
List<String> trimAndAdd(List<String> input, String newItem) {
    int capacity = input.size() + (newItem == null ? 0 : 1);
    List<String> result = new ArrayList<>(capacity);
    for (String s : input) {
        result.add(s.trim());
    }
    if (newItem != null) {
        result.add(newItem.trim());
    }
    return result;
}
```

반환할 리스트의 원소 개수가 미리 정해져 있다면, 이렇게 초기 용량을 지정해주는 게 재할당을 줄여 좋다.

이런 우선순위 실수는 **정적 분석 도구가 비교적 쉽게 잡아낸다.** "비교/조건 연산자의 한쪽 피연산자가 결코 null/특정 값이 될 수 없다"는 패턴을 감지하기 때문이다.

### 가능하면 표준 라이브러리를 쓰자

직접 짜면 이런 미묘한 버그가 끼기 쉬우므로, 아주 간단해 보여도 표준 라이브러리 메서드로 처리하는 걸 권장한다. 예를 들어 동일 문자를 반복해 문자열을 만들 때 루프로 직접 짜지 말고, Java 11부터 추가된 `String.repeat()`를 쓴다.

```java
String line = "=".repeat(10); // "=========="
```

### 실수 방지 가이드

- 조건(삼항) 표현식이 더 큰 표현식의 일부일 땐 **괄호로 묶는다.**
- 우선순위가 헷갈리면 **중간 변수로 표현식을 분할**한다. 짧아지고 의도가 드러난다.
- 직접 구현하기 전에 **표준 라이브러리 메서드**가 있는지 먼저 확인한다.
- 정적 분석기를 켜두면 이런 우선순위 실수를 상당수 미리 걸러준다.
