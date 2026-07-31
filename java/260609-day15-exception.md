# Java 예외 & Spring 트랜잭션 롤백 정책

## 1. 예외 계층 구조

```
Throwable
├── Error                  → JVM 오류, 예외처리 대상 아님
└── Exception
    ├── IOException 등     → Checked 예외 (일반 예외)
    └── RuntimeException   → Unchecked 예외 (실행 예외)
        ├── NullPointerException
        ├── IllegalArgumentException
        └── ArrayIndexOutOfBoundsException
```

---

## 2. Checked vs Unchecked 차이

| 구분 | Checked (일반 예외) | Unchecked (실행 예외) |
|---|---|---|
| 상속 | `Exception` 직접 상속 | `RuntimeException` 상속 |
| 컴파일러 강제 | ✅ `try-catch` 또는 `throws` 필수 | ❌ 없어도 컴파일 OK |
| 발생 시점 | 예측 가능한 외부 상황 (파일 없음, 네트워크 등) | 프로그래머 실수 (null 참조, 배열 범위 초과 등) |

```java
// Checked → throws 없으면 컴파일 에러
public void readFile() throws IOException {
    FileReader fr = new FileReader("test.txt");
}

// Unchecked → throws 없어도 컴파일 OK, 런타임에 터짐
public void doSomething(String s) {
    System.out.println(s.length()); // null이면 NPE
}
```

---

## 3. @Transactional 기본 롤백 정책

| 예외 종류 | 기본 동작 | 이유 |
|---|---|---|
| `RuntimeException` (Unchecked) | ✅ 자동 롤백 | 예상 못한 버그 → 전부 없던 일로 |
| `Error` | ✅ 자동 롤백 | JVM 수준 오류 |
| `Exception` (Checked) | ❌ 롤백 안 함 → **커밋** | 예측된 상황 → DB 작업은 유효하다고 판단 |

> **핵심**: 예외처리 강제 여부가 아니라 **의도의 차이**
> - Checked = "예측된 상황이니 커밋 유지"
> - Unchecked = "예상 못한 오류니 전부 롤백"

---

## 4. 롤백 정책 커스터마이징

```java
// Checked 예외도 롤백하고 싶을 때
@Transactional(rollbackFor = IOException.class)

// RuntimeException인데 롤백 하기 싫을 때 (거의 안 씀)
@Transactional(noRollbackFor = IllegalArgumentException.class)
```

---

## 5. 실무 패턴

- Spring / JPA는 대부분의 Checked 예외를 **Unchecked로 래핑**해서 던짐
  → `SQLException` → `DataAccessException` (RuntimeException)
- 커스텀 예외는 보통 `RuntimeException`을 상속해서 자동 롤백 타게 설계

# System.out.println() 안 쓰는 이유

// System.out → 무조건 출력, 끌 수가 없음
System.out.println("디버그용 출력인데 운영에서도 다 나옴...");

// Logger → 환경별로 on/off 가능
log.debug("개발환경에서만 보임");
log.info("운영에서도 보임");
log.error("에러만 따로 모아서 알림 가능");

// PrintStream.println() 내부
public void println(String x) {
    synchronized (this) {  // ← 여기! 동기라서 멀티스레드 환경에서 병목
        print(x);
        newLine();
    }
}

System.out은 끄지도 못하고, 파일에 안 남고, 느리고, 정보도 부족

# SLF4J 이름의 유래

**Simple Logging Facade for Java**의 약자야.


| 단어 | 의미 |
|---|---|
| **Simple** | 단순한 |
| **Logging** | 로깅 |
| **Facade** | 창구 / 인터페이스 |
| **for Java** | Java용 |



### Facade(퍼사드)가 핵심

SLF4J는 **직접 로그를 기록하는 게 아니라**, 로깅 구현체들 앞에 세워두는 **공통 창구**야.

```
내 코드
  │
  │  log.info("주문 완료")
  ▼
[ SLF4J ]  ← 창구 (인터페이스만 있음)
  │
  ├── Logback  (Spring Boot 기본)
  ├── Log4j2
  └── java.util.logging
```

덕분에 구현체를 바꿔도 **내 코드는 안 바꿔도 됨**.

## 가장 많이 쓰는 구현체: **Logback**

Spring Boot가 **기본으로 내장**하고 있어서 사실상 표준이야.



### 왜 Logback이 기본이냐면

```
spring-boot-starter-web
        │
        └── spring-boot-starter-logging
                    │
                    └── logback-classic  ← 여기 이미 들어있음
```

`spring-boot-starter` 의존성 추가하는 순간 **자동으로 딸려옴**.  
별도 설정 없이 `@Slf4j` 쓰면 Logback이 동작하는 거야.



### 구현체 비교

| | Logback | Log4j2 |
|---|---|---|
| Spring Boot 기본 | ✅ | ❌ (교체 필요) |
| 성능 | 좋음 | 더 좋음 (비동기 특화) |
| 설정 파일 | `logback.xml` | `log4j2.xml` |
| 사용 빈도 | 압도적 다수 | 대용량 트래픽 서비스 |

Log4j2가 성능은 더 좋은데, 굳이 교체할 이유가 없어서 대부분 Logback 그냥 씀.



### 실제로 쓸 때

```java
@Slf4j          // ← SLF4J 인터페이스
@Service
public class OrderService {
    public void order() {
        log.info("주문 완료");  // 실제론 Logback이 처리
    }
}
```

# 전역 예외 처리

ErrorCode  →  에러 종류 정의 (목록표)
CustomException 클래스→  ErrorCode를 들고 예외 던지기
@RestControllerAdvice  →  던진 예외 받아서 JSON 응답

1) ErrorCode로 에러 정의 → 
2) CustomException으로 던지기 → 
3) @RestControllerAdvice로 전역에서 잡아서 JSON 응답! 🎯


# `@RestControllerAdvice` 분해

```
@RestControllerAdvice
    = @ControllerAdvice
    + @ResponseBody
```

그리고 **`@ControllerAdvice`는 AOP의 `@Aspect`를 내부적으로 활용**해.


## AOP 관점에서 보면

```
요청
 │
 ▼
Controller
 │
 ▼
Service ──── 예외 발생! ────────────────────────┐
                                                │
                                    (AOP가 낚아챔)
                                                │
                                                ▼
                                   @RestControllerAdvice
                                   (= @AfterThrowing 역할)
                                                │
                                                ▼
                                          JSON 응답 반환
```

AOP에서 배웠던 **`@AfterThrowing`** 기억나지? 예외 던졌을 때 실행되는 advice.  
`@RestControllerAdvice`가 그 역할을 **Controller 계층 전체에** 적용한 것!


## 근데 완전한 AOP는 아니야

일반 AOP랑 비교하면 차이가 있어:

| | 일반 AOP (`@Aspect`) | `@ControllerAdvice` |
|---|---|---|
| 적용 대상 | 모든 Bean | **Controller 계층만** |
| 구현 방식 | 프록시 | **DispatcherServlet 내부 처리** |
| 포인트컷 지정 | 직접 설정 | 자동 (Controller 예외 전체) |
| 예외 처리 특화 | ❌ | ✅ |


## DispatcherServlet이 중간에서 해주는 것

```
예외 발생
    │
    ▼
DispatcherServlet
    │
    │  "이 예외 처리할 수 있는 HandlerExceptionResolver 있어?"
    ▼
ExceptionHandlerExceptionResolver   ← @ExceptionHandler 찾아줌
    │
    ▼
@RestControllerAdvice 안의 @ExceptionHandler 실행
    │
    ▼
JSON 응답
```

`DispatcherServlet`이 예외를 받아서 **`@ExceptionHandler`가 붙은 메서드를 찾아서** 실행해주는 구조야.


## advice 이름이 붙은 이유

AOP 용어에서 **Advice = "끼어드는 부가 로직"** 이잖아.

```
Controller Advice
= Controller 계층에 끼어드는 부가 로직
= 예외 처리라는 횡단 관심사를 분리
```

이름 자체가 AOP에서 따온 거야!


## 정리

```
@RestControllerAdvice
    ├── 사상(철학)  → AOP (횡단 관심사 분리)
    ├── 실제 구현   → DispatcherServlet + HandlerExceptionResolver
    └── 효과        → Controller 전체에 예외처리 일괄 적용
```

완전한 AOP 프록시 방식은 아니지만, **AOP의 철학을 Controller 예외처리에 특화해서 구현한 것**이라고 보면 돼!


**한 줄 요약**: `@RestControllerAdvice`는 AOP 철학("부가 로직 분리")에서 이름을 따왔고, DispatcherServlet이 예외를 낚아채서 `@ExceptionHandler`로 연결해주는 구조! 🎯

 # Enum 상수
 는 static final 객체로 컴파일되어 Metaspace에 딱 한 번 올라가고, JVM이 싱글톤을 자동 보장해줘! 🎯

 ## 좋은 관찰이야! 하나씩 정리해줄게


## 1. 낯선 문법 설명

```java
INVALID_INPUT(400, "잘못된 입력입니다"),
```

이게 사실 **생성자 호출**이야.

```java
public enum ErrorCode {
    // ↓ 이게
    INVALID_INPUT(400, "잘못된 입력입니다"),

    // ↓ 이거랑 같은 말
    public static final ErrorCode INVALID_INPUT = new ErrorCode(400, "잘못된 입력입니다");

    // Enum 생성자
    ErrorCode(int status, String message) {
        this.status = status;
        this.message = message;
    }
}
```

Enum 특유의 문법이라 낯선 거지, 결국 **생성자에 인자 넘기는 것**이야.


## 2. "빈처럼 관리되는 느낌" 정확해!

```
Spring Bean          Enum 상수
─────────────────────────────────
IoC 컨테이너가 관리   JVM이 관리
앱 시작 시 생성       클래스 로딩 시 생성
싱글톤 보장           싱글톤 보장
어디서든 주입받아 사용  어디서든 가져다 사용
```

관리 주체만 다르고 **"미리 만들어진 객체를 재사용"** 한다는 개념은 같아.


## 3. 메모리 관점 - 이 부분이 핵심!

```java
// CustomException을 new 할 때
throw new CustomException(ErrorCode.USER_NOT_FOUND);
```

메모리에서 일어나는 일:

```
Metaspace
└── ErrorCode.USER_NOT_FOUND  (이미 존재, 새로 생성 안 함)
        │
        │  참조만 전달 (주소값)
        ▼
Heap
└── new CustomException(...)  ← 이것만 새로 생성됨
        │
        └── errorCode 필드  →  USER_NOT_FOUND 주소값 저장
```

```java
@Getter
public class CustomException extends RuntimeException {
    private final ErrorCode errorCode;  // 객체 복사 X, 주소값만 들고 있음

    public CustomException(ErrorCode errorCode) {
        this.errorCode = errorCode;  // 참조 복사
    }
}
```

## 정리

```
ErrorCode Enum 상수  →  Metaspace에 딱 한 번, 앱 종료까지 유지
CustomException      →  throw 할 때마다 Heap에 새로 생성
                         (단, ErrorCode는 참조만 가져다 씀)
```

그래서 네 말이 맞아 — **ErrorCode 객체 자체는 재사용**되고, CustomException만 새로 만들어지는 거야!


**한 줄 요약**: Enum 상수는 주소값만 전달되므로 재사용되고, 실제로 Heap에 새로 생기는 건 CustomException 객체뿐! 🎯

# `ConcurrentHashMap`을 쓰는 이유


### 문제: 일반 `HashMap`은 멀티스레드에서 위험해

```java
Map<String, Integer> map = new HashMap<>();

// 스레드 A
map.put("count", map.get("count") + 1);

// 스레드 B (동시에)
map.put("count", map.get("count") + 1);
```

- **데이터 유실** - 두 스레드가 동시에 write하면 한쪽이 덮어씌워짐
- **무한루프** - Java 7 이하에서 resize 중 circular linked list 발생 가능
- **`ConcurrentModificationException`** - 순회 중 다른 스레드가 수정하면 터짐


### 해결책 비교

```java
// 1. HashMap - 스레드 안전 ❌
Map<String, Integer> map = new HashMap<>();

// 2. Collections.synchronizedMap - 메서드 전체에 락 걸림 (느림)
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());

// 3. ConcurrentHashMap - 부분 락 (빠름) ✅
Map<String, Integer> map = new ConcurrentHashMap<>();
```


### `ConcurrentHashMap`이 빠른 이유 - **Segment 락 (부분 락)**

```
synchronizedMap         ConcurrentHashMap
┌─────────────┐         ┌────┬────┬────┐
│  전체 락 🔒  │         │락🔒│    │    │  ← 버킷 단위로만 락
│  A 작업 중  │         │ A  │ B  │ C  │  ← B, C는 동시 접근 가능
│  B 대기...  │         │    │    │    │
└─────────────┘         └────┴────┴────┘
  한 번에 1개만            여러 스레드 동시 작업 가능
```

Java 8부터는 Segment 대신 **버킷(노드) 단위로 `synchronized`** 적용해서 더 세밀해짐.


### 주의할 점

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ❌ 이건 여전히 안전하지 않음 (get + put이 원자적이지 않음)
map.put("count", map.get("count") + 1);

// ✅ 원자적 연산 제공하는 메서드 써야 함
map.merge("count", 1, Integer::sum);
map.compute("count", (k, v) -> v == null ? 1 : v + 1);
```


**한 줄 요약**: `HashMap`은 멀티스레드에서 터지고, `synchronizedMap`은 너무 느리고, `ConcurrentHashMap`은 **버킷 단위 부분 락**으로 안전하면서도 빠르게 동시성을 해결해줘!

# ENUM과 스레드 세이프

학교 교실에 칠판이 하나 있어요. 선생님이 등교하자마자 칠판에 **"오늘 반장: 김철수"** 라고 딱 한 번 써놓고 나갑니다.

이제 30명의 학생이 우르르 교실로 들어와요. 학생들이 동시에 칠판을 봐도 괜찮을까요? 전혀 문제없죠. 왜냐면 아무도 그 글씨를 지우거나 바꾸지 않으니까요. 그냥 읽기만 하는 거잖아요.

이게 정확히 enum의 thread-safe 원리예요.

- 선생님 = JVM (자바 가상머신)
- 칠판에 쓰는 행위 = 클래스 로딩 (딱 한 번만 일어남)
- "오늘 반장: 김철수" = enum 상수 (`Season.SPRING` 같은 것들)
- 30명의 학생 = 여러 개의 스레드
- 학생들이 동시에 칠판 보는 것 = 스레드들이 동시에 enum에 접근

칠판 내용이 절대 안 바뀌니까 → 누가 언제 봐도 괜찮아요.

반대로 위험한 경우는, 누군가 칠판을 지우고 새로 쓰는 도중에 다른 사람이 보는 거예요. 이걸 `thread-unsafe`라고 해요. 하지만 enum은 한 번 쓰면 절대 안 지우니까 이 문제가 아예 없는 거예요.---

정리하면 세 줄이에요.

1. JVM이 `Season` 클래스를 처음 쓸 때 enum 상수를 딱 한 번 만들어놓음
2. 그 이후로 아무도 그 값을 바꿀 수 없음
3. 바꾸는 사람이 없으니 → 몇 명이 동시에 읽어도 안전

`synchronized`나 `volatile` 같은 동기화 코드가 왜 필요 없냐고 물으면 → "바꾸는 일이 없으니까요" 한 마디면 돼요.

# super() 왜 호출해야 하나?

**핵심 이유: 부모 클래스도 생성자가 있기 때문**

객체를 만들 때 Java는 자식 → 부모 → 조부모 순으로 거슬러 올라가며 **모든 조상 클래스의 초기화**를 보장해야 해.

```
CustomException 생성
    └── RuntimeException 초기화 (super() 호출)
            └── Exception 초기화
                    └── Throwable 초기화  ← 여기서 message, cause 등 저장
```

`Throwable`이 `message`, `cause`, `stackTrace` 같은 핵심 필드를 들고 있어.  
`super(errorCode.getMessage())`를 안 하면 → `getMessage()` 호출 시 `null` 반환됨.


### super() 안 쓰면 어떻게 되나?

```java
// super() 생략하면?
public CustomException(ErrorCode errorCode) {
    // 컴파일러가 자동으로 super() 삽입 → 인자 없는 버전 호출
    this.errorcode = errorCode;
}

// 그러면 이렇게 됨
exception.getMessage(); // → null (message 세팅 안 됨)
```

컴파일 에러는 안 나지만, **부모가 기대하는 초기화가 안 된** 반쪽짜리 객체가 만들어짐.


## Lombok 생성자 쓸 수 없는 이유

결론부터: **이 경우 Lombok 생성자 쓰면 안 됨** ❌

| | Lombok `@RequiredArgsConstructor` | 직접 작성 |
|---|---|---|
| `super()` 호출 | ❌ 불가 | ✅ 가능 |
| `this.field =` 초기화 | ✅ 자동 | ✅ 직접 |

Lombok이 생성하는 코드는 이렇게 생겼어:

```java
// @RequiredArgsConstructor가 만드는 것 (대략)
public CustomException(ErrorCode errorcode) {
    this.errorcode = errorcode;  // super() 호출 없음!
}
```

`super(errorCode.getMessage())`처럼 **부모에게 값을 넘기는 로직**은 Lombok이 알 수 없으니까 생성 자체가 불가능해.


### 그럼 언제 Lombok 쓸 수 있나?

```java
// ✅ 이런 단순한 경우엔 OK
public class UserDto {
    private final String name;
    private final int age;
    // → @RequiredArgsConstructor 쓰면 됨
}

// ❌ 부모 생성자에 값 넘겨야 하면 직접 작성
public class CustomException extends RuntimeException {
    private final ErrorCode errorCode;

    public CustomException(ErrorCode errorCode) {
        super(errorCode.getMessage()); // ← 이 줄 때문에 Lombok 불가
        this.errorCode = errorCode;
    }
}
```

**한 줄 요약:** `super()`에 인자를 넘겨야 하는 순간 = Lombok 포기하고 직접 작성 🙅

## `super(errorCode.getMessage())` 호출 시 내부 흐름

### 호출 체인 따라가기

```
CustomException(errorCode)
    └── super(errorCode.getMessage())  →  RuntimeException(String message)
            └── super(message)         →  Exception(String message)
                    └── super(message) →  Throwable(String message)
                            └── this.detailMessage = message  ← 여기서 저장!
```

`RuntimeException` → `Exception` → `Throwable` 순으로 올라가고,  
결국 **`Throwable`의 생성자**에서 실제 저장이 일어남.


### Throwable 내부 (실제 JDK 코드)

```java
public class Throwable {
    private String detailMessage;  // ← 여기에 저장됨

    public Throwable(String message) {
        fillInStackTrace();          // 스택 트레이스 캡처
        detailMessage = message;     // 메시지 저장
    }

    public String getMessage() {
        return detailMessage;        // 나중에 꺼낼 때 여기서 반환
    }
}
```


### 그래서 실제로 어떻게 쓰이냐

```java
throw new CustomException(ErrorCode.USER_NOT_FOUND);

// 어딘가에서 catch
catch (CustomException e) {
    e.getMessage();  // → ErrorCode.USER_NOT_FOUND.getMessage() 반환
                     //    ex) "유저를 찾을 수 없습니다"
}
```

`super()`에 넘긴 값이 `Throwable.detailMessage`에 저장되고,  
나중에 `e.getMessage()` 호출 시 그게 반환되는 구조야.

### super()에 아무것도 안 넘기면?

```java
public CustomException(ErrorCode errorCode) {
    super(); // 또는 생략
    this.errorCode = errorCode;
}

// 그러면
e.getMessage();  // → null
e.getMessage();  // ErrorCode 정보는 어디에도 안 들어감
```

`Throwable`에 `detailMessage`가 세팅되지 않아서 `null` 반환.  
로그에도 메시지 없이 exception 이름만 찍히게 됨. 😱


**한 줄 요약:** `super(message)` 호출 = 결국 `Throwable.detailMessage = message` 한 줄이 실행되는 것 🎯

## ErrorResponse DTO를 왜 따로 쓰는가?

### 먼저 DTO 없이 처리하면?

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<String> handleCustomException(CustomException e) {
        return ResponseEntity
            .status(e.getErrorCode().getStatus())
            .body(e.getMessage());  // ← 그냥 문자열만 반환
    }
}
```

응답이 이렇게 나옴:
```
"유저를 찾을 수 없습니다"
```


### 근데 실제 서비스에서 프론트가 원하는 건?

```json
{
  "status": 404,
  "code": "USER_NOT_FOUND",
  "message": "유저를 찾을 수 없습니다",
  "timestamp": "2026-06-09T12:00:00"
}
```

프론트 입장에서는 단순 문자열만 받으면:
- 에러 **코드**로 분기 처리를 못 함
- HTTP status만으론 세분화된 에러 구분 불가
- 응답 **구조가 성공/실패 간에 일관성이 없음**


### ErrorResponse DTO 도입

```java
// ErrorResponse.java
@Getter
public class ErrorResponse {
    private final int status;
    private final String code;       // "USER_NOT_FOUND"
    private final String message;    // "유저를 찾을 수 없습니다"

    // ErrorCode로부터 바로 생성
    public static ErrorResponse of(ErrorCode errorCode) {
        return new ErrorResponse(
            errorCode.getStatus().value(),
            errorCode.name(),
            errorCode.getMessage()
        );
    }
}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<ErrorResponse> handleCustomException(CustomException e) {
        ErrorCode errorCode = e.getErrorCode();
        return ResponseEntity
            .status(errorCode.getStatus())
            .body(ErrorResponse.of(errorCode));  // ← 구조화된 JSON
    }
}
```


### 전체 구조 비교

| | DTO 없이 | ErrorResponse DTO 사용 |
|---|---|---|
| 응답 형태 | 문자열 or 랜덤 구조 | 항상 동일한 JSON 구조 |
| 프론트 에러 분기 | status 코드만 가능 | `code` 필드로 세분화 가능 |
| 성공/실패 응답 일관성 | ❌ | ✅ |
| 확장성 | 없음 | timestamp, path 등 추가 쉬움 |


### 최종 흐름 (DTO 포함)

```
throw new CustomException(ErrorCode.USER_NOT_FOUND)
    ↓
GlobalExceptionHandler.handleCustomException()
    ↓
ErrorResponse.of(errorCode)   ← DTO 생성
    ↓
ResponseEntity<ErrorResponse> 반환
    ↓
{"status": 404, "code": "USER_NOT_FOUND", "message": "..."}
```

**한 줄 요약:** ErrorResponse DTO = 프론트와의 **에러 응답 규격 계약서**, 일관된 JSON 구조 보장 📋

## 정적 팩토리 메서드 패턴

### 핵심 개념

`new` 키워드 대신 **이름 있는 static 메서드**로 객체를 만드는 것.

```java
// 일반 생성자
ErrorResponse response = new ErrorResponse(404, "USER_NOT_FOUND", "유저 없음");

// 정적 팩토리 메서드
ErrorResponse response = ErrorResponse.of(errorCode);
```

둘 다 객체 만드는 건 똑같은데, 왜 굳이?

---

### 생성자 vs 정적 팩토리 메서드

**생성자의 한계:**

```java
// 이 둘을 구분할 수 없음 - 이름이 똑같으니까
public ErrorResponse(ErrorCode errorCode) { ... }
public ErrorResponse(String message) { ... }

// 호출할 때 뭘 만드는지 의미가 안 보임
new ErrorResponse(errorCode);  // 뭘 만드는 건지?
new ErrorResponse("직접 메시지");  // 뭘 만드는 건지?
```

**정적 팩토리 메서드:**

```java
// 이름으로 의도가 명확
ErrorResponse.of(errorCode);           // ErrorCode로부터 생성
ErrorResponse.withMessage("직접 메시지"); // 메시지 직접 지정
ErrorResponse.notFound();              // 404 전용
```

---

### 자주 쓰는 네이밍 컨벤션

| 이름 | 의미 | 예시 |
|---|---|---|
| `of` | 여러 매개변수 받아서 생성 | `ErrorResponse.of(errorCode)` |
| `from` | 단일 매개변수 변환 | `LocalDate.from(temporal)` |
| `valueOf` | 값으로부터 생성 | `Integer.valueOf("123")` |
| `getInstance` | 동일 인스턴스 반환 가능 | `Calendar.getInstance()` |
| `create` | 매번 새 인스턴스 보장 | `Array.create()` |

Java 표준 라이브러리도 다 이 패턴임:
```java
List.of(1, 2, 3);
Optional.of(value);
LocalDate.of(2026, 6, 9);
```

---

## @Builder는 정적 팩토리 메서드 패턴이야?

**반만 맞음.** 엄밀히는 **빌더 패턴(Builder Pattern)** 이야.

```java
@Builder
public class ErrorResponse {
    private int status;
    private String code;
    private String message;
}

// 사용할 때
ErrorResponse response = ErrorResponse.builder()  // ← 이게 정적 팩토리 메서드
    .status(404)
    .code("USER_NOT_FOUND")
    .message("유저 없음")
    .build();
```

구조를 보면:

```
ErrorResponse.builder()   ← static 메서드 (정적 팩토리랑 비슷)
    .status(404)          ┐
    .code("USER_NOT_FOUND")┤← Builder 객체의 메서드 체이닝
    .message("유저 없음")  ┘
    .build()              ← 최종 객체 생성
```

`builder()`는 정적 메서드지만, 목적이 **객체를 바로 만드는 게 아니라 Builder를 만드는 것**이라 엄밀히는 다름.

---

### 한눈에 비교

| | 정적 팩토리 메서드 | 빌더 패턴 |
|---|---|---|
| 호출 | `ErrorResponse.of(errorCode)` | `ErrorResponse.builder()...build()` |
| 반환 | 바로 완성된 객체 | Builder 객체 → 최종 객체 |
| 언제 유리 | 매개변수 적고 고정적 | 매개변수 많고 선택적 |
| 가독성 | 이름으로 의도 표현 | 필드명 명시적 |

```java
// 필드 3개 이하 → 정적 팩토리 메서드가 깔끔
ErrorResponse.of(errorCode);

// 필드 많고 일부 선택적 → 빌더가 유리
User.builder()
    .name("김지나")
    .email("...")
    .role(ADMIN)
    .build();
```

**한 줄 요약:** 정적 팩토리 = "이름 있는 생성자", 빌더 = "조립식 생성자", `builder()`는 정적 메서드를 활용하지만 다른 패턴 🏗️

## GlobalExceptionHandler 비교 분석

**코드 1** — 가장 단순한 구조. 로깅도 없고 validation 에러의 필드 메시지를 버리는 게 치명적. 빠른 프로토타입이나 학습용으론 괜찮지만 실제 서비스엔 부족.

```java
@RestControllerAdvice // @RestController에서 발생하는 모든 예외를 가로챈 뒤 처리하는 클래스
public class GlobalExceptionHandler {
  
  //@valid 검증 실패시 발생하는 MethodArgumentNotValidException 예외처리기
  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<ErrorResponse> handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
    ErrorResponse errorResponse = ErrorResponse.of(ErrorCode.INVALID_INPUT_VALUE);

    return new ResponseEntity<>(errorResponse, ErrorCode.INVALID_INPUT_VALUE.getStatus());
  }

  // custom 예외 처리기 - 예외에서 꺼내쓰기
   @ExceptionHandler(CustomException.class)
   public ResponseEntity<ErrorResponse> handleCustomException(CustomException e) {
    ErrorCode errorCode = e.getErrorcode();
    ErrorResponse errorResponse = ErrorResponse.of(errorCode);
    return new ResponseEntity<>(errorResponse, errorCode.getStatus());
   }
  
   // 나머지 모든 예외 처리기
   @ExceptionHandler(Exception.class)
   public ResponseEntity<ErrorResponse> handleException(Exception e){
    ErrorResponse errorResponse = ErrorResponse.of(ErrorCode.INTERNAL_SERVER_ERROR);
    return new ResponseEntity<>(errorResponse, ErrorCode.INTERNAL_SERVER_ERROR.getStatus());
   }
}
```

**코드 2** — 로깅 + 필드 메시지 추출 다 있고, `BusinessException`이라는 네이밍이 가장 의도를 잘 드러냄. 단, 응답 포맷이 정상 응답과 달라서 클라이언트가 두 가지 구조를 처리해야 함.

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .findFirst()
                .map(error -> error.getDefaultMessage())
                .orElse("입력값을 확인해주세요");
        log.warn("Validation failed: {}", message);
        ErrorResponse errorResponse = new ErrorResponse("INVALID_INPUT", message, LocalDateTime.now());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.error("BusinessException occurred: {}", e.getMessage(), e);
        ErrorResponse errorResponse = ErrorResponse.of(e.getErrorCode());
        return ResponseEntity.status(e.getErrorCode().getHttpStatus()).body(errorResponse);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        log.error("Unexpected exception occurred: {}", e.getMessage(), e);
        ErrorResponse errorResponse = ErrorResponse.of(ErrorCode.INTERNAL_SERVER_ERROR);
        return ResponseEntity.status(500).body(errorResponse);
    }
}
```

**코드 3** — `ApiResponse<Void>` 래퍼 덕분에 정상/오류 응답 구조가 동일함. 프론트엔드 입장에서 `response.success ? ... : response.message` 하나로 끝남. 메서드 체이닝(`ResponseEntity.badRequest().body(...)`)도 깔끔.

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(CustomException.class)
  public ResponseEntity<ApiResponse<Void>> handleCustomException(CustomException e) {
    ErrorCode errorCode = e.getErrorCode();
    log.warn("CustomException: {}", errorCode.getMessage());
    return ResponseEntity.status(errorCode.getStatus())
        .body(ApiResponse.error(errorCode.getStatus(), errorCode.getMessage()));
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<ApiResponse<Void>> handleValidationException(
      MethodArgumentNotValidException e) {
    String message =
        e.getBindingResult().getFieldErrors().stream()
            .findFirst()
            .map(error -> error.getDefaultMessage())
            .orElse("잘못된 입력입니다");
    log.warn("Validation failed: {}", message);
    return ResponseEntity.badRequest().body(ApiResponse.error(400, message));
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ApiResponse<Void>> handleException(Exception e) {
    log.error("Unhandled exception", e);
    return ResponseEntity.internalServerError().body(ApiResponse.error(500, "서버 내부 오류입니다"));
  }
}
```

---

### 코드 3이 이기는 결정적 이유

```
정상 응답:  ApiResponse<UserResponse> { success: true, data: {...} }
오류 응답:  ApiResponse<Void>         { success: false, message: "..." }
```

클라이언트가 **하나의 타입**으로 모든 응답을 처리 가능. 코드 1, 2는 오류일 때만 `ErrorResponse` 구조가 오니까 클라이언트 측 분기가 생김.

---

### 코드 3에서 더 개선하면 좋을 것들

```java
// 1. BusinessException으로 이름 바꾸기 (코드 2처럼)
@ExceptionHandler(BusinessException.class)  // CustomException → BusinessException

// 2. validation 에러도 필드 목록 전부 담기 (findFirst 대신)
List<String> errors = e.getBindingResult().getFieldErrors().stream()
        .map(FieldError::getDefaultMessage)
        .collect(Collectors.toList());
// → 어떤 필드들이 틀렸는지 한 번에 다 보내주면 UX 굿

// 3. 서버 에러는 stack trace 포함해서 로깅
log.error("Unhandled exception", e);  // ← 코드 3 이미 잘 하고 있음 ✓
```

**한 줄 요약**: 코드 3은 응답 포맷 통일성이라는 실무 핵심을 잡았고, 거기에 로깅까지 갖췄으니 가장 완성도 높음. 예외 클래스 이름만 `BusinessException`으로 바꾸면 거의 실무 수준.