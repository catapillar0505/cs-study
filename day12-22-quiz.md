# Day 12 ~ 22 복습 퀴즈 (20문제)

> 각 문제의 답은 토글을 클릭하면 확인할 수 있습니다.

---

## Q1. `@SpringBootApplication`은 어떤 어노테이션들의 조합인가?

① `@ComponentScan` + `@EnableAutoConfiguration` + `@Configuration`  
② `@ComponentScan` + `@EnableAutoConfiguration` + `@SpringBootConfiguration`  
③ `@ComponentScan` + `@AutoConfiguration` + `@SpringBootConfiguration`  
④ `@BeanScan` + `@EnableAutoConfiguration` + `@SpringBootConfiguration`

<details>
<summary>정답 확인</summary>

**② `@ComponentScan` + `@EnableAutoConfiguration` + `@SpringBootConfiguration`**

> 함정: ①번처럼 `@Configuration`이라고 착각하기 쉽지만, `@SpringBootConfiguration`이 맞다.  
> `@SpringBootConfiguration`은 Spring의 `@Configuration`을 Spring Boot용으로 확장한 것이다.  
> 기능은 거의 동일하지만 이름이 다르다.

</details>

---

## Q2. 생성자 주입에서 `final` 키워드를 사용하는 이유로 **틀린** 것은?

① 객체 생성 후 의존성 교체를 막아 불변성을 보장한다  
② 멀티스레드 환경에서 안전하다  
③ `@Autowired` 필드 주입으로도 `final` 필드에 값을 주입할 수 있다  
④ 의존성 빈을 찾지 못하면 객체 자체가 생성되지 않아 NPE를 방지한다

<details>
<summary>정답 확인</summary>

**③ `@Autowired` 필드 주입으로도 `final` 필드에 값을 주입할 수 있다**

> 함정: 필드 주입은 리플렉션을 사용하는데, 리플렉션은 객체 **생성 이후**에 값을 꽂아넣는 방식이라  
> `final`(생성자에서만 초기화 가능)과 충돌한다.  
> **`final`이 붙은 필드에는 생성자 주입만 가능**하며, 필드 주입(`@Autowired`)은 사용할 수 없다.

</details>

---

## Q3. HTTP 상태코드에 대한 설명으로 올바른 것은?

① 401은 "권한 없음", 403은 "로그인 필요"를 의미한다  
② 204는 "요청이 잘못됨"을 의미한다  
③ 302는 "영구 이동"을 의미하여 브라우저가 URL을 캐싱한다  
④ 401은 "신원 확인 안 됨(로그인 필요)", 403은 "신원은 알지만 권한 없음"을 의미한다

<details>
<summary>정답 확인</summary>

**④ 401은 "신원 확인 안 됨(로그인 필요)", 403은 "신원은 알지만 권한 없음"을 의미한다**

> 함정: 401과 403의 의미가 자주 뒤바뀐다.  
> - 401 → "너 누구야?" (인증 실패, 로그인 필요)  
> - 403 → "너 누군지 알아, 근데 안 돼" (인가 실패, 권한 없음)  
> - 204는 "삭제 성공", 302는 "임시 이동"이다.

</details>

---

## Q4. Spring MVC에서 모든 요청을 하나의 서블릿이 받아 적절한 컨트롤러로 분배하는 패턴은?

① Proxy 패턴  
② Strategy 패턴  
③ Front Controller 패턴  
④ Singleton 패턴

<details>
<summary>정답 확인</summary>

**③ Front Controller 패턴**

> `DispatcherServlet`이 모든 요청의 관문 역할을 하며,  
> 어느 Controller로 보낼지 결정(Handler Mapping)하는 구조를 **Front Controller 패턴**이라 한다.

</details>

---

## Q5. `@RestController`에 대한 설명으로 올바른 것은?

① `@Controller` + `@RequestBody`의 조합이다  
② `@Controller` + `@ResponseBody`의 조합이다  
③ `@Service` + `@ResponseBody`의 조합이다  
④ `@Controller`만으로도 JSON을 반환할 수 있어 동일하다

<details>
<summary>정답 확인</summary>

**② `@Controller` + `@ResponseBody`의 조합이다**

> 함정: `@RequestBody`와 헷갈리기 쉽다.  
> - `@RequestBody` → 요청 Body(JSON)를 Java 객체로 **역직렬화**  
> - `@ResponseBody` → Java 객체를 JSON으로 **직렬화**하여 응답 Body에 담음  
> `@RestController`는 클래스 전체에 `@ResponseBody`를 적용한 것과 동일하다.

</details>

---

## Q6. 다음 요청에 적합한 어노테이션은?

```
GET /users?name=Kim&age=20
```

① `@PathVariable`  
② `@RequestBody`  
③ `@RequestParam`  
④ `@ModelAttribute`

<details>
<summary>정답 확인</summary>

**③ `@RequestParam`**

> 함정: `@ModelAttribute`도 쿼리 스트링을 받을 수 있지만,  
> REST API에서 **조건으로 검색/필터링**할 때는 `@RequestParam`이 적합하다.  
> - `@PathVariable` → URL 경로 (`/users/1`)  
> - `@RequestParam` → 쿼리 스트링 (`?name=Kim`)  
> - `@RequestBody` → HTTP Body(JSON)

</details>

---

## Q7. `@NotNull`, `@NotEmpty`, `@NotBlank`에 대한 설명으로 **틀린** 것은?

① `@NotNull`은 `null`만 걸러내며, 빈 문자열(`""`)은 통과된다  
② `@NotEmpty`는 `null`과 빈 문자열(`""`)을 모두 걸러낸다  
③ `@NotBlank`는 `null`, `""`, `" "`(공백)까지 모두 걸러낸다  
④ `@NotNull`은 빈 문자열과 공백 문자열까지 모두 걸러낸다

<details>
<summary>정답 확인</summary>

**④ `@NotNull`은 빈 문자열과 공백 문자열까지 모두 걸러낸다**

> 함정: `@NotNull`은 **`null`만** 검증한다.  
> `""` (빈 문자열)와 `" "` (공백)은 `@NotNull` 검증을 **통과**해버린다.  
> - `@NotNull` : null 차단  
> - `@NotEmpty` : null + "" 차단  
> - `@NotBlank` : null + "" + " " 차단 (가장 강력)

</details>

---

## Q8. 게시글 작성 후 새로고침으로 인한 중복 INSERT를 방지하는 패턴으로 올바른 것은?

① POST → Forward → GET (PFG 패턴)  
② POST → Redirect → GET (PRG 패턴)  
③ GET → Redirect → POST (GRP 패턴)  
④ Forward 대신 Redirect를 쓸 필요가 없다, 동일하다

<details>
<summary>정답 확인</summary>

**② POST → Redirect → GET (PRG 패턴)**

> **PRG(Post → Redirect → Get) 패턴**: INSERT 후 Redirect하면 브라우저 URL이 GET으로 바뀌어  
> F5(새로고침)를 눌러도 GET 요청만 재실행된다.  
> Forward를 쓰면 URL이 여전히 POST `/board/write`로 남아 새로고침 시 중복 INSERT가 발생한다.

</details>

---

## Q9. RESTful API에서 **멱등성(Idempotent)이 없는** HTTP 메서드는?

① GET  
② PUT  
③ DELETE  
④ POST

<details>
<summary>정답 확인</summary>

**④ POST**

> 함정: DELETE도 멱등성이 있다고 착각하기 쉬운데, **있다**.  
> 같은 리소스를 여러 번 삭제해도 결과는 "삭제됨"으로 동일하다.  
> **POST(등록)만 멱등성이 없다.** 같은 요청을 100번 보내면 100개가 생성된다.  
> (결제, 주문 서비스가 POST를 쓰는 이유)

</details>

---

## Q10. Spring AOP에서 **Joinpoint**에 대한 설명으로 올바른 것은?

① Joinpoint는 실제로 Advice가 적용되는 지점만을 의미한다  
② Spring AOP에서 Joinpoint는 메서드 실행 시점이 **유일하다**  
③ Pointcut은 Advice가 적용 가능한 모든 지점을 의미한다  
④ Aspect는 Pointcut만으로 구성된다

<details>
<summary>정답 확인</summary>

**② Spring AOP에서 Joinpoint는 메서드 실행 시점이 **유일하다****

> 함정: AspectJ는 필드 접근, 생성자 호출 등 다양한 Joinpoint를 지원하지만,  
> **Spring AOP는 메서드 실행 시점만 Joinpoint로 지원한다.**  
> - Joinpoint: Advice를 적용할 수 있는 모든 지점  
> - Pointcut: Joinpoint 중 **실제로 선택된** 지점  
> - Aspect: Pointcut + Advice를 묶은 클래스

</details>

---

## Q11. `@Transactional`의 기본 롤백 정책으로 올바른 것은?

① `IOException`(Checked 예외) 발생 시 기본적으로 롤백된다  
② `RuntimeException`(Unchecked 예외) 발생 시 기본적으로 커밋된다  
③ `IOException`(Checked 예외) 발생 시 기본적으로 **커밋**된다  
④ 모든 예외에 대해 기본적으로 롤백된다

<details>
<summary>정답 확인</summary>

**③ `IOException`(Checked 예외) 발생 시 기본적으로 **커밋**된다**

> 함정: 예외가 발생하면 무조건 롤백될 것 같지만 그렇지 않다.  
> - `RuntimeException` / `Error` → 기본 **롤백**  
> - `Exception`(Checked) → 기본 **커밋** (예측된 상황이므로 DB 작업은 유효하다고 판단)  
> Checked 예외도 롤백하려면 `@Transactional(rollbackFor = IOException.class)` 명시 필요.

</details>

---

## Q12. AOP의 Advice 실행 시점 중 **메서드 실행 전후를 모두 감싸며** 가장 많이 사용되는 것은?

① `@Before`  
② `@After`  
③ `@AfterReturning`  
④ `@Around`

<details>
<summary>정답 확인</summary>

**④ `@Around`**

> `@Around`는 메서드 실행 **전후**를 모두 제어할 수 있어 가장 강력하고 많이 사용된다.  
> - `@Before` → 메서드 실행 전  
> - `@After` → 메서드 실행 후 (성공/실패 무관)  
> - `@AfterReturning` → 정상 반환 후  
> - `@Around` → 전후 모두, 실행 자체를 제어 가능

</details>

---

## Q13. `CustomException`을 만들 때 Lombok의 `@RequiredArgsConstructor`를 **사용할 수 없는** 이유는?

① `CustomException`이 `final` 클래스라서  
② Lombok이 `super(errorCode.getMessage())`처럼 부모 생성자에 값을 넘기는 코드를 생성할 수 없어서  
③ `RuntimeException`은 `@RequiredArgsConstructor`와 호환이 안 돼서  
④ `@Getter`와 `@RequiredArgsConstructor`는 같이 쓸 수 없어서

<details>
<summary>정답 확인</summary>

**② Lombok이 `super(errorCode.getMessage())`처럼 부모 생성자에 값을 넘기는 코드를 생성할 수 없어서**

> Lombok이 생성하는 생성자에는 `super()` 호출이 없다.  
> `RuntimeException`의 메시지를 세팅하려면 `super(errorCode.getMessage())`를 직접 호출해야 하는데,  
> Lombok은 이 로직을 알 수 없으므로 생성자를 **직접 작성**해야 한다.

</details>

---

## Q14. `@RestControllerAdvice`에 대한 설명으로 **틀린** 것은?

① `@ControllerAdvice` + `@ResponseBody`의 조합이다  
② `DispatcherServlet`이 예외를 잡아 `@ExceptionHandler`로 연결해준다  
③ `@Aspect`를 사용한 일반 AOP와 완전히 동일한 방식으로 동작한다  
④ AOP의 철학(횡단 관심사 분리)을 Controller 예외 처리에 특화한 것이다

<details>
<summary>정답 확인</summary>

**③ `@Aspect`를 사용한 일반 AOP와 완전히 동일한 방식으로 동작한다**

> `@RestControllerAdvice`는 AOP의 **철학**을 따르지만, 구현 방식이 다르다.  
> - 일반 AOP: 프록시 객체로 동작  
> - `@ControllerAdvice`: `DispatcherServlet` + `HandlerExceptionResolver`로 동작  
> 또한 일반 AOP는 모든 Bean에 적용되지만, `@ControllerAdvice`는 **Controller 계층에만** 적용된다.

</details>

---

## Q15. Java의 `LocalDateTime`을 MySQL에 매핑할 때 올바른 타입은?

① `TIMESTAMP`  
② `DATETIME`  
③ `DATE`  
④ `VARCHAR`

<details>
<summary>정답 확인</summary>

**② `DATETIME`**

> 함정: `TIMESTAMP`와 혼동하기 쉽다.  
> - `LocalDateTime` → 타임존 정보가 **없음** → `DATETIME` 매핑  
> - `TIMESTAMP` → DB 타임존에 따라 값이 변환됨 (생성일시, 수정일시에 사용)  
> - `DATETIME` 범위: 1000년 ~ 9999년 / `TIMESTAMP` 범위: 1970년 ~ 2038년

</details>

---

## Q16. JPA의 **1차 캐시**에 대한 설명으로 올바른 것은?

① 애플리케이션 전체에서 공유되는 글로벌 캐시다  
② `EntityManagerFactory` 단위로 하나만 존재한다  
③ 같은 `EntityManager`(트랜잭션) 내에서만 유효하다  
④ `@Cacheable` 어노테이션으로 활성화해야 한다

<details>
<summary>정답 확인</summary>

**③ 같은 `EntityManager`(트랜잭션) 내에서만 유효하다**

> 함정: 1차 캐시는 **애플리케이션 전체 캐시가 아니다**.  
> - **1차 캐시**: `EntityManager`(영속성 컨텍스트) 단위, 트랜잭션이 끝나면 사라짐  
> - **2차 캐시**: `EntityManagerFactory` 단위, 애플리케이션 전체 공유  
> 같은 트랜잭션에서 동일 ID를 두 번 조회하면 두 번째는 DB 조회 없이 1차 캐시에서 반환된다.

</details>

---

## Q17. JPA에서 `@ManyToOne`의 기본 FetchType은?

① `LAZY`  
② `EAGER`  
③ `DEFAULT`  
④ 설정하지 않으면 에러가 발생한다

<details>
<summary>정답 확인</summary>

**② `EAGER`**

> 함정: `@OneToMany`의 기본값은 `LAZY`지만, **`@ManyToOne`의 기본값은 `EAGER`다.**  
> 이 때문에 `@ManyToOne`을 아무 설정 없이 쓰면 연관 엔티티를 항상 함께 조회한다.  
> 성능 문제를 방지하려면 `@ManyToOne(fetch = FetchType.LAZY)`로 **명시적으로 LAZY 설정**을 권장한다.

</details>

---

## Q18. JPA에서 `@ManyToOne`과 `@OneToMany` 관계에서 FK(외래키)는 어느 쪽 테이블에 위치하는가?

① `@OneToMany`가 붙은 쪽 (부모, 1쪽)  
② `@ManyToOne`이 붙은 쪽 (자식, N쪽)  
③ 양쪽 모두에 FK가 생성된다  
④ 중간 테이블에 FK가 생성된다

<details>
<summary>정답 확인</summary>

**② `@ManyToOne`이 붙은 쪽 (자식, N쪽)**

> DB 관점에서 FK는 항상 **N(여러 개) 쪽** 테이블에 존재한다.  
> 예: `PostComment`가 `@ManyToOne`으로 `Post`를 참조 → `PostComment` 테이블에 `post_id` FK  
> `@ManyToOne` = "나는 N이고 FK를 가진 자식"  
> 중간 테이블은 `@ManyToMany` 관계에서 생성된다.

</details>

---

## Q19. Spring Security에서 **인증(Authentication)**과 **인가(Authorization)**의 순서로 올바른 것은?

① 인가 먼저 체크, 실패하면 인증 요청  
② 인증과 인가는 동시에 처리된다  
③ 인증 먼저, 인증 성공 후 인가 체크  
④ 순서는 없으며 설정에 따라 달라진다

<details>
<summary>정답 확인</summary>

**③ 인증 먼저, 인증 성공 후 인가 체크**

> 반드시 **인증(Authentication) → 인가(Authorization)** 순서여야 한다.  
> - 인증: "이 사람이 누구인가?" (로그인)  
> - 인가: "이 사람이 이걸 해도 되나?" (권한 체크)  
> 인증도 안 된 사람에게 인가를 체크하는 건 의미가 없다.  
> Spring Security 필터 체인도 이 순서로 처리된다.

</details>

---

## Q20. JWT(JSON Web Token)의 Payload에 대한 설명으로 올바른 것은?

① Payload는 서버의 비밀키로 암호화되어 있어 클라이언트가 읽을 수 없다  
② Payload는 Base64로 **인코딩**되어 있어 누구나 디코딩해서 내용을 볼 수 있다  
③ Payload는 서버만 가지고 있는 정보라 안전하다  
④ Payload에는 비밀번호를 저장해도 안전하다

<details>
<summary>정답 확인</summary>

**② Payload는 Base64로 **인코딩**되어 있어 누구나 디코딩해서 내용을 볼 수 있다**

> 함정: JWT가 "서명(Signature)"되어 있어 안전하다고 생각하기 쉽지만,  
> **Payload는 암호화가 아닌 Base64 인코딩**이라 누구나 디코딩 가능하다.  
> JWT의 보안은 **위조 불가** (서명 검증)에 있는 것이지, **내용 숨김** (암호화)이 아니다.  
> 따라서 Payload에는 **민감 정보(비밀번호 등)를 절대 담으면 안 된다.**  
> JWT 구조: `Header(Base64)`.`Payload(Base64)`.`Signature(비밀키 서명)`

</details>

---

> 총 20문제 | Day12(Q1~4) · Day13(Q5~8) · Day14(Q9~10) · Day15(Q11~14) · Day16(Q15) · Day18(Q16~18) · Day21(Q19) · Day22(Q20)
