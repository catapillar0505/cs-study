> [← ipc 목차](README.md)

# OpenFeign — 선언적 HTTP

## 한 줄로

**인터페이스만 선언하면 구현체를 스프링이 런타임에 만들어 준다.** URL 조립·JSON 변환·주소 찾기를 대신 해주므로, 남의 서비스를 부르는 코드가 그냥 메서드 호출처럼 보인다.

## 문제부터 — 손으로 짜면 이렇게 됩니다

게시판 서비스가 회원 서비스에서 작성자 이름을 가져와야 한다고 합시다.

```java
RestClient client = RestClient.create();
UserResponse user = client.get()
    .uri("http://localhost:8081/api/users/" + userId)   // 주소를 코드가 알아야 함
    .retrieve()
    .body(UserResponse.class);                          // JSON → 객체 변환도 내가 지정
```

무엇이 불편한가:

- **주소가 코드에 박힙니다.** 서버 IP가 바뀌거나 3대로 늘면 코드를 고쳐야 합니다
- **URL 문자열 조립**을 매번 손으로 합니다 (오타가 나기 쉽습니다)
- **상태코드 분기, JSON 변환** 코드가 호출할 때마다 반복됩니다

## "선언적(Declarative)"이 무슨 뜻인가

프로그래밍에는 두 가지 스타일이 있습니다.

| | 명령형(Imperative) | 선언적(Declarative) |
|---|---|---|
| 적는 것 | **어떻게** 할지 절차를 다 적음 | **무엇을** 원하는지만 적음 |
| 나머지는 | 내가 다 함 | **프레임워크가 알아서** |
| 일상 비유 | "왼쪽으로 200m, 우회전, 300m…" | **"서울역으로 가주세요"** |

Feign은 선언적입니다. **"이런 API를 부르고 싶다"고 인터페이스로 선언만** 하면 끝입니다.

```java
@FeignClient(name = "user-service")        // ← 상대 서비스의 "이름"
public interface UserClient {

  @GetMapping("/api/users/{userId}")       // ← 상대방 API 명세를 그대로 베껴 씀
  UserResponse getUser(@PathVariable("userId") Long userId);
}
```

**본문이 없습니다. 구현 클래스도 없습니다.** 그런데 동작합니다.

## 어떻게 동작하나 — 프록시

**프록시(Proxy)** 는 "대리인"이라는 뜻입니다.
진짜 객체인 척 앞에 서 있다가, 호출이 들어오면 **가로채서 자기 일을 하는** 객체입니다.

```
   내 코드가 부르는 것          실제로 일하는 것
   userClient.getUser(1L)  ──▶  [Feign 프록시]
                                    │ ① 이름으로 실제 주소 찾기
                                    │ ② HTTP 요청 만들어 전송
                                    │ ③ JSON 응답 → 객체로 변환
                                    ▼
                                 UserResponse 반환
```

순서:

1. `@EnableFeignClients`가 `@FeignClient` 인터페이스들을 **찾아냅니다**
2. 스프링이 **런타임에 프록시 객체를 만들어** 빈으로 등록합니다
3. 호출이 오면 프록시가 위 ①②③을 대신합니다

> **JPA에서 `JpaRepository` 인터페이스만 선언해도 `save()`가 동작하는 것**과
> **완전히 같은 원리**입니다. 본문 없는 인터페이스에 구현체가 런타임에 붙는 것.

## 주소를 어떻게 찾나 — 두 가지 방식

```java
@FeignClient(name = "user-service")                          // ① 이름으로 찾기
@FeignClient(name = "market-service", url = "http://localhost:8081")  // ② 주소 직접 지정
```

| | 동작 | 언제 |
|---|---|---|
| ① `name`만 | **서비스 디스커버리(Eureka)에 이름으로 물어봄.** 인스턴스가 여러 개면 로드밸런서가 하나 고름 | 실전 구성 |
| ② `url` 있음 | 디스커버리를 **건너뛰고** 그 주소로 바로 붙음 | 학습·테스트, 외부 API |

**①이 핵심입니다.** `@FeignClient`의 `name` 값이 **Eureka에 등록된 서비스 이름**과 이어지는 지점이,
디스커버리와 Feign이 만나는 접점입니다. → [msa/02 서비스 디스커버리](../02-서비스-디스커버리.md)

## 가장 헷갈리는 지점 — 같은 어노테이션, 반대 역할

```java
// 서버 쪽 (user-service) — "이 URL로 요청이 오면 내가 처리한다"
@RestController
public class UserController {
  @GetMapping("/api/users/{userId}")
  public UserResponse getUser(@PathVariable Long userId) { ... }
}

// 클라이언트 쪽 (board-service) — "이 URL로 요청을 보내겠다"
@FeignClient(name = "user-service")
public interface UserClient {
  @GetMapping("/api/users/{userId}")
  UserResponse getUser(@PathVariable("userId") Long userId);
}
```

**`@GetMapping`이 양쪽에 똑같이 있는데 역할이 정반대입니다.**

| | 뜻 |
|---|---|
| `@RestController` 안의 `@GetMapping` | **서버를 만드는 코드** — "이 주소를 열어둔다" |
| `@FeignClient` 안의 `@GetMapping` | **요청을 만드는 설계도** — "이 주소로 보낸다" |

> ⚠️ **그래서 경로가 글자 하나까지 정확히 같아야 합니다.**
> 상대가 `/api/users/{id}`인데 내가 `/api/user/{id}`라고 쓰면 **404**입니다.
> 그리고 그 사실은 **실행해봐야** 압니다(런타임). 컴파일러는 아무것도 잡아주지 못합니다.
> 이게 [gRPC](03-grpc와-protobuf.md)와 대비되는 REST의 근본적인 약점입니다.

## 에러는 어떻게 넘어오나 — ErrorDecoder

없는 사용자 `999`를 조회하면 상대는 **404**를 돌려줍니다. 그런데 Feign의 기본 동작은
**2xx가 아닌 모든 응답을 `FeignException` 하나로 뭉뚱그립니다.**

그러면 호출하는 쪽이 "이게 404라서 실패한 건가, 500이라 실패한 건가"를 매번 뒤져야 합니다.

**`ErrorDecoder`는 "남의 HTTP 상태코드"를 "내 도메인 예외"로 옮기는 통역사**입니다.

```
 상대 서비스           Feign                     내 비즈니스 로직
   404 응답  ──▶  ErrorDecoder가 번역  ──▶  UserNotFoundException
                  "404면 UserNotFound"        ↑ 의미가 있는 이름
```

번역을 한 곳에 모아두면, 비즈니스 로직은 **`UserNotFoundException`이라는 의미 있는 이름**만
신경 쓰면 되고, HTTP 상태코드를 몰라도 됩니다.

> **통신 디버깅이 안 될 때 1번 수단**: Feign 로그 레벨을 `FULL`로 올리면
> 요청/응답의 헤더·바디·상태코드가 전부 콘솔에 찍힙니다.
> 단, Feign은 `DEBUG` 레벨로 로그를 남기므로 **해당 패키지의 로깅 레벨도 같이 낮춰야** 보입니다.

## 흔한 오해

**"Feign이 HTTP 라이브러리다"**
아닙니다. Feign은 **껍데기(추상화)** 이고, 실제 전송은 그 아래의 HTTP 클라이언트가 합니다.
Feign이 하는 일은 "인터페이스를 보고 요청을 조립해 넘기는 것"입니다.

**"`@FeignClient`의 `name`은 아무 이름이나 써도 된다"**
`url`이 있을 때만 그렇습니다. `url`이 없으면 그 이름으로 **디스커버리에 조회**하므로
등록된 서비스 이름과 정확히 같아야 합니다.

---

[← 서비스 간 통신이란](01-서비스-간-통신이란.md) | [gRPC와 Protobuf →](03-grpc와-protobuf.md)
