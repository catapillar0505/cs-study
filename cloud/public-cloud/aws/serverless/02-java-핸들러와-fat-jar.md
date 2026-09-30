> [← serverless 목차](README.md) · [aws](../README.md)

# Java 핸들러와 Fat JAR

## 한 줄로

**`RequestHandler<입력, 출력>`을 구현한 클래스 하나**가 Lambda 함수다. 들어온 JSON은 입력 타입으로 자동 변환되고, 반환값은 JSON으로 바뀌어 나간다. 콘솔에 올릴 때는 **의존성까지 한 JAR에 묶어야** 한다.

## 가장 작은 핸들러 — `lambda-basic`

```java
public class HelloHandler implements RequestHandler<Map<String, String>, String> {
  @Override
  public String handleRequest(Map<String, String> input, Context context) {
    LambdaLogger logger = context.getLogger();
    logger.log("Input: " + input);

    String name = input.getOrDefault("name", "홍길동");
    return "Hello " + name;
  }
}
```

| 부분 | 역할 |
|---|---|
| `RequestHandler<I, O>` | Lambda 런타임이 호출할 **약속된 인터페이스** (`aws-lambda-java-core`) |
| `I` = `Map<String, String>` | 들어온 JSON을 **이 타입으로 역직렬화**해서 넣어줌 |
| `O` = `String` | 반환값을 **JSON으로 직렬화**해서 호출자에게 돌려줌 |
| `Context` | 요청 ID, 남은 실행 시간, 함수 이름, **로거** |
| `LambdaLogger` | 찍은 로그가 **CloudWatch Logs**로 감 |

콘솔 "테스트"에 이런 이벤트를 넣으면

```json
{ "name": "김철수" }
```

결과는 `"Hello 김철수"` — 문자열도 JSON으로 나가므로 **따옴표가 붙어서** 보입니다.

### 핸들러 이름 형식

콘솔의 "런타임 설정 → 핸들러"에 적는 값입니다.

```
com.example.lambda.HelloHandler::handleRequest
└──── 패키지 + 클래스 ────┘  └ 메서드 ┘
```

**패키지명까지 정확히** 적어야 합니다. 틀리면 `ClassNotFoundException`이 납니다.

## 입력을 DTO로 받기 — `lambda-calculator`

`Map` 대신 **record**를 입력 타입으로 쓰면 필드가 명확해집니다.

```java
public record CalcRequest(String num1, String num2, String op) { }

public class CalculatorHandler implements RequestHandler<CalcRequest, String> {
  public String handleRequest(CalcRequest input, Context context) {
    String calculatorName = System.getenv("CALCULATOR_NAME");   // 환경 변수
    if (calculatorName == null || calculatorName.isBlank()) {
      calculatorName = "Lambda Calculator";
    }
    double num1 = Double.parseDouble(input.num1());
    ...
  }
}
```

```json
{ "num1": "10", "num2": "3", "op": "*" }
```

JSON의 **키 이름 = record 필드 이름**이면 자동으로 채워집니다. 없는 키는 `null`이 됩니다.

### 환경 변수

코드를 다시 빌드하지 않고 **설정만 바꾸고 싶은 값**을 넣는 곳입니다. (콘솔 → 구성 → 환경 변수)

- `System.getenv("CALCULATOR_NAME")` 으로 읽는다
- **없을 때의 기본값을 꼭 둔다** — 위 코드처럼
- 버킷 이름, 테이블 이름처럼 **환경마다 다른 값**이 주로 들어간다 → SAM에서는 템플릿이 자동으로 채워준다 ([06](06-실습-s3-presigned-url.md))

> 비밀번호·API 키는 환경 변수에 평문으로 두지 말고 Secrets Manager / Parameter Store를 씁니다.

## 왜 Fat JAR인가

### 문제

`./gradlew build`로 만든 **보통 JAR에는 내 클래스만** 들어 있습니다.

```
lambda-basic.jar
└── com/example/lambda/HelloHandler.class
    (aws-lambda-java-core 는 없음!)
```

내 컴퓨터에서는 Gradle이 의존성을 classpath에 넣어주지만, **Lambda에는 내가 올린 파일만 있습니다.** 의존성이 늘어나면 `NoClassDefFoundError`가 납니다.

### 해결 — Shadow 플러그인

```groovy
plugins {
  id 'java'
  id 'com.gradleup.shadow' version '9.6.1'   // 의존성을 JAR 안에 풀어서 합쳐줌
}

dependencies {
  implementation 'com.amazonaws:aws-lambda-java-core:1.4.0'
}

// Shadow 결과물은 기본 이름이 *-all.jar → 접미사를 없애서 보통 이름으로
tasks.named('shadowJar') {
  archiveClassifier.set('')
}

// build 하면 shadowJar 도 같이 돌게
tasks.named('build') {
  dependsOn tasks.named('shadowJar')
}

// 한글 로그/문자열이 깨지지 않게
tasks.withType(JavaCompile).configureEach {
  options.encoding = 'UTF-8'
}
```

```
lambda-basic-0.0.1-SNAPSHOT.jar   ← build/libs/ 에 생김, 이걸 콘솔에 업로드
├── com/example/lambda/HelloHandler.class
└── com/amazonaws/services/lambda/runtime/...   ← 의존성이 같이 들어감
```

| | 보통 JAR | Fat JAR (Shadow) |
|---|---|---|
| 내용 | 내 코드만 | 내 코드 + 모든 의존성 |
| 실행 조건 | 의존성을 따로 classpath에 | **이 파일 하나면 끝** |
| Lambda 콘솔 업로드 | 의존성 쓰면 실패 | ✓ |

> **SAM을 쓰면 Shadow가 필요 없습니다.** `sam build`가 의존성을 `lib/` 폴더에 모아서 같이 패키징해주기 때문입니다. → [04](04-sam과-cloudformation.md)

## 흔한 실수 — 실습 코드에서 나온 것들

### ① 복사-붙여넣기 버그

실습 계산기 코드는 **모든 연산자가 덧셈**을 하고 있었습니다.

```java
case "+": result = num1 + num2; break;
case "-": result = num1 + num2; break;   // ← 뺄셈이어야 함
case "*": result = num1 + num2; break;   // ← 곱셈이어야 함
case "/": result = num1 + num2; break;   // ← 나눗셈이어야 함
```

콘솔 테스트를 `+`로만 해보면 **절대 안 잡힙니다.** 연산자마다 테스트 이벤트를 하나씩 만들어 두는 게 좋습니다.

Java 21이면 switch 식으로 쓰는 게 이런 실수를 줄입니다.

```java
double result = switch (op) {
  case "+" -> num1 + num2;
  case "-" -> num1 - num2;
  case "*" -> num1 * num2;
  case "/" -> {
    if (num2 == 0) throw new IllegalArgumentException("0으로 나누기 금지");
    yield num1 / num2;
  }
  default -> throw new IllegalArgumentException("잘못된 연산자: " + op);
};
```

### ② 예외를 문자열로 삼키면 Lambda는 "성공"으로 본다

```java
} catch (Exception e) {
  return "알 수 없는 오류 발생: " + e.getMessage();   // 정상 반환
}
```

| | 예외를 던짐 | 문자열로 반환 |
|---|---|---|
| Lambda가 보는 결과 | **실패** (`errorMessage`, `errorType`) | **성공** |
| CloudWatch 오류 지표 | 올라감 | **안 올라감** |
| 재시도·알림 | 동작 | **동작 안 함** |

사용자에게 보여줄 메시지와 **시스템이 알아야 할 실패**는 구분해야 합니다. HTTP로 노출할 때는 **상태 코드로** 구분합니다. → [03](03-api-gateway와-프록시-통합.md)

---

[← 서버리스와 Lambda](01-서버리스와-lambda.md) | [API Gateway와 프록시 통합 →](03-api-gateway와-프록시-통합.md)
