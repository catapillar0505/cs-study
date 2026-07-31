# Spring Security 면접 질문 & 답변

> 실무에서 실제로 묻는 질문 중심입니다.
> 각 질문을 클릭하면 답변이 펼쳐집니다.

## 목차

- [기본 개념](#기본-개념)
- [필터 & 아키텍처](#필터--아키텍처)
- [인증 흐름](#인증-흐름)
- [세션 & 상태 관리](#세션--상태-관리)
- [인가 & 권한](#인가--권한)
- [보안 설정](#보안-설정)

## 기본 개념

<details>
<summary>Spring Security란 무엇인가요?</summary>

Spring 기반 애플리케이션에서 **인증(Authentication)** 과 **인가(Authorization)** 를 담당하는 보안 프레임워크입니다.

직접 구현하기 어려운 보안 기능들을 제공합니다.

- 로그인/로그아웃 처리
- 세션 및 토큰 관리
- 비밀번호 암호화
- CSRF, XSS, 클릭재킹 방어
- URL 및 메서드 단위 접근 제어

핵심 동작 원리는 **FilterChain** 입니다. 요청이 Controller에 도달하기 전에 여러 필터를 거치며, 인증/인가에 실패하면 Controller까지 도달하지 못합니다.

</details>

<details>
<summary>인증(Authentication)과 인가(Authorization)의 차이점은 무엇인가요?</summary>

```
인증 (Authentication): "당신이 누구인지 증명하세요"
  → 로그인 과정. 아이디/비밀번호가 맞는지 확인
  → 실패 시 401 Unauthorized 응답

인가 (Authorization): "당신이 이걸 해도 되는지 확인합니다"
  → 권한 체크. 관리자만 접근 가능한 페이지 등
  → 실패 시 403 Forbidden 응답
```

**순서가 중요합니다.** 반드시 인증 먼저, 그 다음 인가입니다.
인증되지 않은 사용자에게 인가를 체크하는 것은 의미가 없습니다.

회사 비유: 사원증으로 건물에 들어가는 것이 **인증**, 임원실은 임원만 들어갈 수 있는 것이 **인가**입니다.

</details>

<details>
<summary>401과 403의 차이는 무엇인가요?</summary>

| 상태코드 | 의미 | 발생 상황 |
|---------|------|----------|
| **401 Unauthorized** | 인증 안 됨 | 로그인하지 않은 상태에서 보호된 리소스 요청 |
| **403 Forbidden** | 인가 실패 | 로그인은 했지만 해당 리소스에 접근 권한이 없음 |

Spring Security에서는 각각 다르게 처리합니다.

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint(...)  // 401 처리 — 로그인 안 된 상태
    .accessDeniedHandler(...)       // 403 처리 — 권한 없는 상태
);
```

</details>

## 필터 & 아키텍처

<details>
<summary>톰캣, 서블릿 컨테이너, DispatcherServlet의 관계를 설명해 주세요.</summary>

**톰캣**은 **서블릿 컨테이너**입니다. HTTP 요청을 받아서 서블릿에 전달하는 역할을 합니다.

Spring Boot를 실행하면 내장 톰캣이 뜨고, 그 위에 `DispatcherServlet`이 **딱 하나** 올라갑니다.

```
[톰캣 - 서블릿 컨테이너]
    └─ DispatcherServlet (1개)  ← 모든 요청이 여기로 집결
            ├─ HandlerMapping    (어느 Controller로 보낼지 결정)
            ├─ HandlerAdapter    (Controller 실행)
            └─ ViewResolver      (응답 뷰 결정)
```

전통 Java EE에서는 URL마다 서블릿을 따로 만들었지만,
Spring MVC는 모든 요청을 DispatcherServlet 하나가 받아서 내부에서 Controller로 라우팅합니다.
이것이 **프론트 컨트롤러 패턴**입니다.

Filter는 DispatcherServlet 바깥에 있기 때문에 Spring Bean 세계에 진입하기도 전에 동작합니다.

</details>

<details>
<summary>Filter, Interceptor, AOP의 차이점과 각각 어떤 경우에 사용하나요?</summary>

```
[클라이언트]
    ↓
[Filter]          ← Servlet 레벨. DispatcherServlet 바깥
    ↓
[DispatcherServlet]
    ↓
[Interceptor]     ← Spring MVC 레벨. Controller 직전/직후
    ↓
[Controller]
    ↓
[AOP]             ← 메서드 레벨. 비즈니스 로직 전후
    ↓
[Service / Repository]
```

| 구분 | 동작 위치 | 주요 사용 사례 |
|------|----------|--------------|
| **Filter** | Servlet 컨테이너 | 인증/인가, 인코딩 설정, CORS |
| **Interceptor** | Spring MVC | 로그인 체크, URL 접근 제어, 실행 시간 측정 |
| **AOP** | Spring Bean 메서드 | 트랜잭션, 메서드 단위 로깅, `@PreAuthorize` |

**실무 판단 기준:**
- Spring Security의 인증/인가 → Filter (DispatcherServlet 이전에 처리해야 하므로)
- 인코딩, CORS → Filter (요청 Body를 파싱하기 전에 처리해야 하므로)
- 로그인 여부 확인, URL 접근 제어 → Interceptor
- 메서드 단위 권한 체크, 트랜잭션 → AOP

</details>

<details>
<summary>인코딩 설정을 왜 Filter에서 처리하나요?</summary>

인코딩은 요청/응답 데이터를 어떤 문자 집합(Charset)으로 해석할지 정하는 것입니다.

```
클라이언트가 UTF-8로 "안녕"을 보냄
    ↓
서버가 ISO-8859-1(기본값)로 읽으면 → "???" (한글 깨짐)
```

Request Body를 읽기 **전에** 인코딩을 고정해야 깨지지 않습니다.
Interceptor나 Controller에서 설정하면 이미 Body를 읽은 뒤라 타이밍을 놓칩니다.

Filter가 요청 처리의 가장 앞에 있기 때문에 인코딩 설정은 Filter에서 합니다.
Spring Boot는 `CharacterEncodingFilter`를 자동으로 등록하므로 직접 만들 일은 거의 없습니다.

</details>

<details>
<summary>CORS란 무엇이고, 왜 Filter에서 처리해야 하나요?</summary>

**CORS(Cross-Origin Resource Sharing)** 는 다른 출처(Origin)에서 오는 HTTP 요청을 브라우저가 허용할지 결정하는 보안 정책입니다.

```
출처(Origin) = 프로토콜 + 도메인 + 포트

http://localhost:3000  (React)
http://localhost:8080  (Spring)
→ 포트가 다르면 다른 출처 → 브라우저가 요청을 차단
```

브라우저는 실제 요청 전에 `OPTIONS` 메서드로 **Preflight 요청**을 먼저 보냅니다.

```
브라우저 → OPTIONS /api/login  ("이 출처에서 POST 해도 됩니까?")
서버     → "응, localhost:3000은 허용해"
브라우저 → POST /api/login     (실제 요청 전송)
```

이 Preflight 요청은 Spring Security의 인증 필터보다 **먼저** 처리되어야 합니다.
인증 필터가 Preflight를 인증 실패로 막아버리면 실제 요청 자체가 날아가지 않기 때문입니다.
그래서 CORS는 **Filter 레벨**에서 처리합니다.

</details>

<details>
<summary>Spring Security의 FilterChain 동작 방식을 설명해 주세요.</summary>

Spring Security는 `SecurityFilterChain`이라는 필터 묶음을 FilterChain에 끼워 넣습니다.

```
클라이언트 요청
    ↓
[SecurityFilterChain]
    ├─ SecurityContextPersistenceFilter   (세션에서 인증 정보 복원)
    ├─ UsernamePasswordAuthenticationFilter (폼 로그인 처리)
    ├─ BasicAuthenticationFilter          (Basic 인증 처리)
    ├─ ExceptionTranslationFilter         (보안 예외 → HTTP 응답 변환)
    └─ FilterSecurityInterceptor          (최종 인가 결정)
    ↓
DispatcherServlet → Controller
```

요청이 Controller에 도달하기 전에 필터들이 순서대로 실행됩니다.
인증/인가 실패 시 Controller까지 도달조차 하지 못하고 응답이 반환됩니다.

</details>

## 인증 흐름

<details>
<summary>UsernamePasswordAuthenticationToken이란 무엇인가요?</summary>

로그인 요청에서 아이디와 비밀번호를 담는 객체입니다. 두 가지 상태가 있습니다.

```java
// 미인증 상태 — 로그인 요청 시 생성. 그냥 credentials를 담은 상자
new UsernamePasswordAuthenticationToken(username, password)
// isAuthenticated() == false

// 인증 완료 상태 — 인증 성공 후 Provider가 생성해 반환
new UsernamePasswordAuthenticationToken(principal, credentials, authorities)
// isAuthenticated() == true
```

이 토큰을 `AuthenticationManager.authenticate(token)`에 넘기면,
내부에서 DB 조회, 비밀번호 검증이 일어나고 인증 완료 상태의 토큰이 반환됩니다.

</details>

<details>
<summary>폼 로그인 방식과 REST API 방식에서 AuthenticationManager가 호출되는 시점이 다른 이유를 설명해 주세요.</summary>

**방식 1: 폼 로그인 — FilterChain에서 인증**

`UsernamePasswordAuthenticationFilter`가 `/login` POST 요청을 FilterChain 단계에서 가로챕니다.
DispatcherServlet과 Controller에는 도달하지 않습니다.

```
Client → POST /login
    ↓
[FilterChain]
    └─ UsernamePasswordAuthenticationFilter 가 가로챔
            ↓  ← 여기서 AuthenticationManager 호출
        인증 성공/실패 핸들러 → 응답 반환
```

**방식 2: REST API — Controller에서 인증 (우리가 쓰는 방식)**

Security 필터들이 `/login`을 통과시키고, Controller까지 도달한 후
Controller 코드 안에서 `authenticationManager.authenticate()`를 직접 호출합니다.

```
Client → POST /login
    ↓
[FilterChain] → 통과
    ↓
[DispatcherServlet] → [Controller]
    └─ 여기서 AuthenticationManager 직접 호출 ← 이 시점에 인증 시작
```

JSON 기반 REST API는 필터가 아닌 Controller에서 처리하는 방식이 일반적입니다.

</details>

<details>
<summary>UserDetails와 UserDetailsService의 역할을 설명해 주세요.</summary>

**UserDetails**: Spring Security가 사용자 정보를 다루는 표준 인터페이스입니다.
우리가 만든 `User` 엔티티를 이 인터페이스로 감싸서 Spring Security에 전달해야 합니다.

```java
public class CustomUserDetails implements UserDetails {
    private final User user;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole()));
    }

    @Override public String getPassword() { return user.getPassword(); }
    @Override public String getUsername() { return user.getEmail(); }
    @Override public boolean isEnabled() { return user.isActive(); }
    // ... 계정 만료, 잠금 여부 등
}
```

**UserDetailsService**: 인증 흐름에서 DB 조회를 담당합니다.
`loadUserByUsername()`은 우리가 직접 호출하지 않고, `DaoAuthenticationProvider`가 인증 과정에서 자동으로 호출합니다.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("사용자 없음"));
        return new CustomUserDetails(user);
    }
}
```

</details>

<details>
<summary>AuthenticationManager와 AuthenticationProvider의 관계를 설명해 주세요.</summary>

```
AuthenticationManager (인터페이스) ← 인증 요청을 받는 창구
    └─ ProviderManager (구현체)
            ├─ DaoAuthenticationProvider       ← DB 기반 인증
            │       ├─ UserDetailsService 호출  (사용자 조회)
            │       └─ PasswordEncoder 사용     (비밀번호 검증)
            ├─ OAuth2LoginAuthenticationProvider ← 소셜 로그인
            └─ (커스텀 Provider 추가 가능)
```

`AuthenticationManager`는 인증 요청을 받으면 자신이 가진 `Provider` 목록을 순서대로 돌며 처리 가능한 Provider에 위임합니다.

일반 로그인은 `DaoAuthenticationProvider`가 처리하며, 내부에서 `UserDetailsService`로 사용자를 조회하고 `PasswordEncoder`로 비밀번호를 대조합니다.

</details>

<details>
<summary>BCrypt를 사용하는 이유는 무엇인가요?</summary>

비밀번호를 평문으로 저장하면 DB 유출 시 바로 노출됩니다.
BCrypt는 단방향 해시 함수로, 암호화된 값에서 원래 비밀번호를 복원할 수 없습니다.

**BCrypt의 특징:**
- 같은 비밀번호를 `encode()`해도 매번 **다른 결과**가 나옵니다 (Salt 자동 포함)
- 레인보우 테이블 공격(미리 계산된 해시 역추적 공격)을 방어합니다
- 비교는 반드시 `matches(raw, encoded)`로 해야 합니다. `==` 비교 불가

```java
// 회원가입: 저장 전 암호화
user.setPassword(passwordEncoder.encode(rawPassword));

// 로그인: 평문과 암호화된 값 비교
boolean isMatch = passwordEncoder.matches(rawPassword, encodedPassword);
```

</details>

## 세션 & 상태 관리

<details>
<summary>SecurityContextHolder란 무엇이고 왜 ThreadLocal을 사용하나요?</summary>

`SecurityContextHolder`는 현재 인증된 사용자 정보를 보관하는 공간입니다.

```
SecurityContextHolder
    └─ SecurityContext
            └─ Authentication
                    ├─ Principal (UserDetails — 사용자 정보)
                    ├─ Credentials (비밀번호, 인증 후 보통 삭제)
                    └─ Authorities (권한 목록: ROLE_USER, ROLE_ADMIN)
```

기본적으로 **ThreadLocal**을 사용합니다. ThreadLocal은 스레드마다 독립된 저장 공간을 가집니다.
덕분에 동시에 수천 명의 사용자가 요청을 보내더라도 서로의 인증 정보가 섞이지 않습니다.

각 요청은 별도 스레드에서 처리되고, 요청이 끝나면 SecurityContext가 초기화됩니다.

```java
// 어디서든 현재 로그인 사용자 꺼내기
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
UserDetails user = (UserDetails) auth.getPrincipal();
```

</details>

<details>
<summary>세션과 쿠키의 차이점은 무엇인가요? 각각 어디에 저장되나요?</summary>

| 구분 | 저장 위치 | 특징 |
|------|----------|------|
| **세션(Session)** | 서버 메모리 | 실제 데이터(인증 정보 등)를 서버에 보관 |
| **쿠키(Cookie)** | 클라이언트 브라우저 | 세션 ID를 담아 브라우저에 저장, 요청마다 자동 첨부 |

**동작 방식:**
```
1. 로그인 성공 → 서버: 세션 생성, 세션 ID 발급
2. 서버 → 브라우저: 세션 ID를 쿠키(JSESSIONID)에 담아 응답
3. 이후 모든 요청 → 브라우저: 쿠키를 자동으로 요청에 포함
4. 서버: 쿠키의 세션 ID로 서버의 세션 데이터 조회 → 사용자 식별
```

서버는 세션 ID만 브라우저에 발급하고, 실제 인증 정보는 서버 메모리에 보관하기 때문에 상대적으로 안전합니다.

</details>

<details>
<summary>SecurityContextHolder에 저장했는데 왜 세션에도 따로 저장해야 하나요?</summary>

`SecurityContextHolder`는 **현재 요청 스레드(ThreadLocal)** 에만 살아있습니다.
같은 요청 안에서는 인증 정보가 유지되지만, 요청이 끝나면 사라집니다.

다음 요청이 왔을 때 "이 사람이 로그인한 사람"임을 알려면,
HTTP 세션에도 SecurityContext를 저장해야 합니다.

```java
// SecurityContextHolder에만 저장 → 현재 요청에서만 유효, 다음 요청 시 로그아웃됨
SecurityContextHolder.setContext(context);

// 세션에도 명시적으로 저장 → 이후 요청에서도 인증 상태 유지
securityContextRepository.saveContext(context, request, response);  // 이 줄이 핵심
```

다음 요청이 오면 `SecurityContextPersistenceFilter`가 세션에서 SecurityContext를 꺼내서 복원합니다.

</details>

<details>
<summary>다중 서버(Scale-out) 환경에서 세션을 어떻게 관리하나요?</summary>

**문제:** 서버가 여러 대일 때 세션이 각 서버 메모리에 따로 저장됩니다.

```
사용자가 서버 A에서 로그인 → 세션이 서버 A에만 저장
다음 요청이 서버 B로 라우팅됨 → 서버 B에 세션 없음 → 로그아웃됨
```

**해결책: Redis를 공용 세션 스토리지로 사용**

```
[로드 밸런서]
    ├── 서버 A ──┐
    ├── 서버 B ──┼──→ [Redis] ← 모든 서버가 공유
    └── 서버 C ──┘
```

Spring Session + Redis를 사용하면 세션이 자동으로 Redis에 저장되며,
어느 서버로 요청이 라우팅되어도 동일한 세션 데이터를 조회할 수 있습니다.

```java
@Configuration
@EnableRedisHttpSession
public class SessionConfig {
    // 이 설정 하나로 세션이 Redis에 저장됨
}
```

Redis를 사용하는 이유: 인메모리 데이터베이스로 속도가 빠르고, TTL(만료 시간) 설정이 간편하며, 여러 서버가 쉽게 공유할 수 있습니다.

</details>

## 인가 & 권한

<details>
<summary>URL 단위 인가와 메서드 단위 인가(@PreAuthorize)의 차이는 무엇인가요?</summary>

**URL 단위 인가** — SecurityFilterChain에서 설정. 필터 레벨에서 동작.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/**").authenticated()
    .anyRequest().permitAll()
);
```

**메서드 단위 인가** — `@PreAuthorize`로 Controller/Service 메서드에 직접 선언. AOP로 동작.

```java
@GetMapping("/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public List<User> getAllUsers() { ... }

// 복잡한 조건도 가능 (본인 글만 수정)
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public void updatePost(@PathVariable Long userId) { ... }
```

**차이점:**
- URL 단위: 경로 패턴 기반. 설정이 한 곳에 모임. 단순 접근 제어에 적합.
- 메서드 단위: 표현식으로 세밀한 조건 가능. 비즈니스 로직과 인가 조건이 함께 보임.

사용하려면 `@EnableMethodSecurity` 어노테이션이 필요합니다.

</details>

<details>
<summary>hasRole()과 hasAuthority()의 차이는 무엇인가요?</summary>

```java
hasRole("ADMIN")        // 내부적으로 "ROLE_ADMIN"과 비교 (ROLE_ prefix 자동 추가)
hasAuthority("ADMIN")   // "ADMIN" 그대로 비교 (prefix 없음)
hasAuthority("ROLE_ADMIN") // "ROLE_ADMIN" 그대로 비교
```

`hasRole()`은 편의 메서드로, `ROLE_` prefix를 자동으로 붙여서 비교합니다.
`hasAuthority()`는 전달한 문자열 그대로 비교합니다.

실무에서는 보통 DB에 권한을 `ADMIN` 또는 `ROLE_ADMIN`으로 저장하고,
`UserDetails.getAuthorities()`에서 `ROLE_` prefix를 붙여서 반환합니다.

```java
return List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole()));
// 이후 hasRole("ADMIN") 또는 hasAuthority("ROLE_ADMIN") 둘 다 동작
```

</details>

## 보안 설정

<details>
<summary>CSRF란 무엇이고, REST API에서 비활성화해도 되는 이유는 무엇인가요?</summary>

**CSRF(Cross-Site Request Forgery)**: 악의적인 사이트에서 사용자의 브라우저를 통해 우리 서버로 의도하지 않은 요청을 보내는 공격입니다.

```
사용자가 우리 서버에 로그인한 상태
    ↓
악의적인 사이트(malicious.com) 방문
    ↓
malicious.com이 우리 서버로 요청을 자동 전송
    ↓
브라우저가 쿠키(세션 ID)를 자동 첨부 → 서버는 정상 요청으로 오해
```

**CSRF 방어**: 서버가 발급한 CSRF 토큰을 요청에 포함시켜, 우리 사이트에서 보낸 요청임을 검증합니다.

**REST API에서 비활성화해도 되는 이유:**
- REST API는 보통 `Content-Type: application/json` 요청을 받습니다
- 악의적인 사이트에서는 이 Content-Type으로 cross-origin 요청을 보내려면 Preflight를 통과해야 하고, CORS 설정에서 막힙니다
- 또한 JWT 같은 토큰 인증은 Authorization 헤더를 사용하므로 쿠키 자동 첨부 문제가 없습니다

```java
http.csrf(csrf -> csrf.disable()); // REST API + JWT 방식에서 일반적
```

</details>

<details>
<summary>Spring Security에서 로그아웃 시 무엇을 처리해야 하나요?</summary>

로그아웃 시 서버 측에서 다음 세 가지를 모두 처리해야 합니다.

```java
http.logout(logout -> logout
    .invalidateHttpSession(true)   // 1. 서버의 세션 데이터 삭제
    .deleteCookies("JSESSIONID")   // 2. 브라우저의 세션 ID 쿠키 삭제
    .clearAuthentication(true)     // 3. SecurityContext 초기화
);
```

하나라도 빠지면 문제가 생깁니다.
- 세션만 무효화하고 쿠키를 삭제 안 하면, 브라우저가 여전히 만료된 세션 ID를 보냅니다.
- SecurityContext만 초기화하고 세션을 무효화 안 하면, 다음 요청에서 세션으로 인증이 복원됩니다.

</details>

<details>
<summary>Spring Security 설정에서 .permitAll()과 .authenticated()의 차이는 무엇인가요?</summary>

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()    // 누구나 접근 가능 (비로그인 포함)
    .requestMatchers("/api/**").authenticated()   // 로그인한 사람만 (권한 무관)
    .requestMatchers("/admin/**").hasRole("ADMIN") // ADMIN 권한을 가진 로그인 사용자만
    .anyRequest().permitAll()                     // 위에 해당 안 되는 모든 경로
);
```

주의: `requestMatchers`는 **위에서 아래로 순서대로** 매칭됩니다.
더 구체적인 경로를 위에 배치해야 합니다.

```java
// 잘못된 예 — anyRequest().permitAll()이 먼저 매칭되어 아래 규칙이 무시될 수 있음
.anyRequest().permitAll()
.requestMatchers("/admin/**").hasRole("ADMIN")  // 이 줄은 의미 없음

// 올바른 예 — 구체적인 규칙을 먼저
.requestMatchers("/admin/**").hasRole("ADMIN")
.anyRequest().permitAll()
```

</details>

<details>
<summary>@AuthenticationPrincipal이란 무엇인가요?</summary>

Controller에서 현재 로그인한 사용자 정보를 꺼낼 때,
`SecurityContextHolder`를 직접 사용하지 않고 파라미터로 바로 받을 수 있게 해주는 어노테이션입니다.

```java
// SecurityContextHolder 직접 사용 — 번거로움
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
CustomUserDetails userDetails = (CustomUserDetails) auth.getPrincipal();

// @AuthenticationPrincipal 사용 — 간결
@GetMapping("/my/profile")
public ResponseEntity<?> getProfile(
        @AuthenticationPrincipal CustomUserDetails userDetails) {
    User user = userDetails.getUser();
    return ResponseEntity.ok(UserDto.from(user));
}
```

Spring Security가 SecurityContext에서 Principal을 꺼내서 파라미터로 주입해 줍니다.
로그인하지 않은 상태로 접근하면 `null`이 주입되므로 `permitAll()` 경로에서 사용 시 null 체크가 필요합니다.

</details>
