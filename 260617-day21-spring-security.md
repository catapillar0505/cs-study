# Spring Security 완전 정복

## 목차

1. [Spring Security가 하는 일](#1-spring-security가-하는-일)
2. [인증 vs 인가 — 가장 먼저 구분해야 할 개념](#2-인증-vs-인가)
3. [필터 체인 — Spring Security의 심장](#3-필터-체인--spring-security의-심장)
4. [필터 vs 인터셉터 vs AOP — 언제 무엇을 쓰나?](#4-필터-vs-인터셉터-vs-aop)
5. [로그인 요청의 시작 — Authentication Token](#5-로그인-요청의-시작--authentication-token)
6. [UserDetails & UserDetailsService — 사용자 정보 구조](#6-userdetails--userdetailsservice)
7. [비밀번호 암호화 — PasswordEncoder](#7-비밀번호-암호화--passwordencoder)
8. [AuthenticationManager & Provider — 인증 처리 중심](#8-authenticationmanager--provider)
9. [SecurityContext — 인증 성공 후 정보 보관함](#9-securitycontext--인증-성공-후-정보-보관함)
10. [세션 인증과 토큰 인증](#10-세션-인증과-토큰-인증)
11. [역할(Role) 기반 인가](#11-역할role-기반-인가)
12. [다중 서버 환경에서의 세션 확장 전략](#12-다중-서버-환경에서의-세션-확장-전략)
13. [CORS & CSRF 설정](#13-cors--csrf-설정)
14. [실무에서 자주 마주치는 패턴](#14-실무에서-자주-마주치는-패턴)

## 1. Spring Security가 하는 일

Spring Security는 **보안을 애플리케이션 레이어에서 처리**해 주는 프레임워크입니다.

직접 구현하면 까다로운 것들을 대신 해줍니다:

| 기능 | 설명 |
|------|------|
| 인증(Authentication) | 이 사람이 누구인가? (로그인) |
| 인가(Authorization) | 이 사람이 이걸 해도 되나? (권한 체크) |
| 세션/토큰 관리 | 로그인 상태 유지 |
| CSRF 방어 | 악의적인 사이트 간 요청 차단 |
| 비밀번호 암호화 | BCrypt 등으로 안전하게 저장 |
| XSS/클릭재킹 방어 | HTTP 보안 헤더 자동 설정 |

## 2. 인증 vs 인가

이 두 개념은 Spring Security 전체의 기반입니다. 절대 헷갈리면 안 됩니다.

```
인증 (Authentication): "당신이 누구인지 증명하세요"
  → 로그인 과정. 아이디/비밀번호 확인

인가 (Authorization): "당신이 이 리소스에 접근할 권한이 있나요?"
  → 관리자만 볼 수 있는 페이지, 본인 글만 수정 가능 등
```

**실생활 비유:**
- 회사 건물에 들어갈 때 사원증으로 **인증** → "나 직원이에요"
- 임원실 문은 임원만 들어갈 수 있음 → **인가** → "당신은 임원이 아니군요, 출입 불가"

**순서가 중요합니다**: 반드시 인증 먼저, 그 다음 인가입니다.
인증도 안 된 사람에게 인가를 체크하는 건 의미가 없습니다.

## 3. 필터 체인 — Spring Security의 심장

Spring Security의 핵심 동작 방식입니다.

```
클라이언트 요청
    ↓
[FilterChain] ← Spring Security가 여기에 필터를 쭉 끼워 넣음
    │
    ├─ SecurityContextPersistenceFilter   (세션에서 인증 정보 복원)
    ├─ UsernamePasswordAuthenticationFilter (폼 로그인 처리)
    ├─ BasicAuthenticationFilter          (Basic 인증 처리)
    ├─ ExceptionTranslationFilter         (보안 예외 → HTTP 응답 변환)
    └─ FilterSecurityInterceptor          (최종 인가 결정)
    ↓
DispatcherServlet (Spring MVC 시작)
    ↓
Controller
```

> **핵심**: 요청이 Controller에 도달하기 **전에** 필터들이 먼저 가로챕니다.
> 인증/인가에 실패하면 Controller까지 도달조차 못합니다.

**SecurityFilterChain 설정 예시 (최신 방식)**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()   // 누구나 접근 가능
                .requestMatchers("/api/admin/**").hasRole("ADMIN") // 어드민만
                .anyRequest().authenticated()                    // 나머지는 로그인 필요
            )
            .formLogin(Customizer.withDefaults())  // 기본 로그인 폼
            .logout(Customizer.withDefaults());    // 기본 로그아웃

        return http.build();
    }
}
```

## 4. 필터 vs 인터셉터 vs AOP

실무에서 "어디에 공통 로직을 넣을까?" 고민할 때 반드시 알아야 하는 개념입니다.

### 먼저 알아야 할 것: 톰캣과 DispatcherServlet의 관계

Spring Boot를 실행하면 내장 **톰캣(Tomcat)** 이 뜹니다.
톰캣은 **서블릿 컨테이너**입니다 — HTTP 요청을 받아서 서블릿에 넘겨주는 역할을 합니다.

여기서 핵심 사실:

> **톰캣 위에 DispatcherServlet은 딱 하나만 올라갑니다.**

```
[톰캣 - 서블릿 컨테이너]
    └─ DispatcherServlet (1개)  ← Spring MVC의 "프론트 컨트롤러"
            ├─ HandlerMapping    (어느 Controller로 보낼지 결정)
            ├─ HandlerAdapter    (Controller 실행)
            └─ ViewResolver      (응답 뷰 결정)
```

전통적인 Java EE에서는 서블릿을 여러 개 만들고 URL마다 따로 매핑했습니다.
Spring MVC는 이 방식을 바꿔서, 모든 요청을 DispatcherServlet **하나**가 받아서
내부적으로 적절한 Controller로 라우팅합니다. 이것이 **프론트 컨트롤러 패턴**입니다.

**Filter는 이 DispatcherServlet 바깥**에 있습니다.
즉, Spring의 ApplicationContext(Bean 세계)에 진입하기도 전에 동작합니다.
Spring Security의 필터들이 인증/인가를 처리할 수 있는 이유가 바로 이것입니다.

```
요청 흐름:

[클라이언트]
    ↓
[톰캣 - 서블릿 컨테이너]
    ↓
[Filter]          ← Servlet 레벨. DispatcherServlet 바깥. Spring Bean 세계 밖
    ↓
[DispatcherServlet]  ← 톰캣에 딱 1개. 여기서부터 Spring MVC 세계
    ↓
[Interceptor]     ← Spring MVC 레벨. Controller 직전/직후
    ↓
[Controller]
    ↓
[AOP]             ← 메서드 레벨. 비즈니스 로직 전후
    ↓
[Service / Repository]
```

### 각각 언제 쓰나?

| 구분 | 접근 가능 범위 | 주요 사용 사례 |
|------|---------------|---------------|
| **Filter** | HttpServletRequest/Response 전체 | 인증/인가, 인코딩 설정, CORS |
| **Interceptor** | Controller 메서드 정보, ModelAndView | 로그인 체크, 특정 URL 접근 제어, 실행 시간 측정 |
| **AOP** | 메서드 파라미터, 반환값, 예외 | 트랜잭션, 메서드 단위 로깅, 권한 체크(`@PreAuthorize`) |

### Filter 사용 사례 부연 설명

**인코딩(Encoding) 설정을 Filter에서 하는 이유**

인코딩이란 요청/응답 데이터를 어떤 문자 집합(Charset)으로 해석할지 정하는 것입니다.
한글이 깨지는 문제의 원인이 여기에 있습니다.

```
클라이언트가 UTF-8로 "안녕"을 보냄
    ↓
서버가 ISO-8859-1(기본값)로 읽으면 → "???" (깨짐)
    ↓
Filter에서 가장 먼저 UTF-8로 강제 설정해야
Request Body를 읽기 전에 인코딩을 고정할 수 있음
```

Filter가 아닌 Interceptor나 Controller에서 설정하면 이미 Body를 읽은 뒤라
타이밍을 놓칩니다. 그래서 가장 앞에 있는 **Filter에서 처리**합니다.

Spring Boot는 `CharacterEncodingFilter`를 자동으로 등록하기 때문에
직접 짤 일은 거의 없지만, 왜 Filter인지 이유를 알고 있어야 합니다.

**CORS를 Filter에서 처리하는 이유**

CORS(Cross-Origin Resource Sharing)란 **다른 출처(Origin)에서 오는 요청을 허용할지 결정하는 브라우저 보안 정책**입니다.

```
출처(Origin) = 프로토콜 + 도메인 + 포트
  http://localhost:3000  ← React 개발 서버
  http://localhost:8080  ← Spring 서버

→ 포트가 다르면 다른 출처 → 브라우저가 요청을 막음
```

브라우저는 실제 요청 전에 `OPTIONS` 메서드로 **사전 요청(Preflight)** 을 먼저 보냅니다.

```
[브라우저] → OPTIONS /api/login → [서버]
              "이 출처에서 POST 해도 됩니까?"

[서버] → "응, localhost:3000은 허용해"
              ↓
[브라우저] → POST /api/login → [서버]  (실제 요청)
```

이 Preflight 요청은 Spring Security의 인증 필터보다 **먼저 처리**되어야 합니다.
인증 필터가 Preflight를 막아버리면 실제 요청 자체가 아예 날아가지 않기 때문입니다.
그래서 CORS 처리는 **Filter 레벨**에서 합니다.

**실무 판단 기준:**
- 보안(인증/인가) → **Filter** (Spring Security가 여기서 동작)
- 인코딩, CORS → **Filter** (요청 파싱 전 가장 먼저 처리해야 하기 때문)
- URL 기반 접근 제어, 로그인 체크 → **Interceptor**
- 메서드 단위 권한 체크, 트랜잭션 → **AOP**

## 5. 로그인 요청의 시작 — Authentication Token

로그인 요청이 들어왔을 때 Spring Security는 가장 먼저 무엇을 하는지 봅니다.

### 인증의 출발점: UsernamePasswordAuthenticationToken

클라이언트가 POST /login으로 아이디와 비밀번호를 보내면,
이 두 값을 `UsernamePasswordAuthenticationToken`이라는 객체에 담습니다.

```java
// 아직 인증되지 않은 상태의 토큰 — 그냥 "이 사람이 이걸 들고 왔어요" 수준
UsernamePasswordAuthenticationToken token =
    new UsernamePasswordAuthenticationToken(username, password);
```

이 토큰은 두 가지 상태가 있습니다:

```
[미인증 상태]  new UsernamePasswordAuthenticationToken(username, password)
    → 인증 전. 그냥 credentials를 담은 상자
    → isAuthenticated() == false

[인증 완료 상태]  new UsernamePasswordAuthenticationToken(principal, credentials, authorities)
    → 인증 성공 후 Provider가 만들어 반환
    → isAuthenticated() == true
```

### 누가 이 토큰을 처리하나?

`AuthenticationManager.authenticate(token)`에 이 토큰을 넘깁니다.

```
UsernamePasswordAuthenticationToken (미인증)
    ↓
AuthenticationManager.authenticate()
    ↓
DaoAuthenticationProvider
    ↓
UserDetailsService.loadUserByUsername()  → DB에서 사용자 조회
    ↓
PasswordEncoder.matches()               → 비밀번호 대조
    ↓
UsernamePasswordAuthenticationToken (인증 완료) 반환
```

### 두 가지 진입 방식 — AuthenticationManager는 어디서 호출되나?

여기가 많이 헷갈리는 포인트입니다. **방식에 따라 AuthenticationManager가 호출되는 위치가 다릅니다.**

**방식 1: 필터가 처리 (폼 로그인)**

`UsernamePasswordAuthenticationFilter`가 `/login` POST 요청을 FilterChain 단계에서 가로챕니다.
Controller까지 아예 도달하지 않습니다.

```
Client → POST /login
    ↓
[FilterChain]
    └─ UsernamePasswordAuthenticationFilter 가 /login 요청 가로챔
            ↓ ← 여기서 AuthenticationManager 호출
        AuthenticationManager.authenticate()
            ↓
        인증 성공/실패 핸들러 → 응답 반환
        (DispatcherServlet, Controller 안 거침)
```

```java
http.formLogin(Customizer.withDefaults()); // 이걸로 끝, 필터가 알아서 처리
```

**방식 2: Controller가 처리 (React + REST API 방식 — 우리가 쓰는 방식)**

Security 필터들이 `/login`을 그냥 통과시킵니다.
FilterChain → DispatcherServlet → Controller 순서로 도달한 뒤,
**Controller 코드 안에서** `authenticationManager.authenticate()`를 직접 호출합니다.

```
Client → POST /login
    ↓
[FilterChain] → 통과 (인증 필터가 /login을 막지 않음)
    ↓
[DispatcherServlet]
    ↓
[Controller]
    └─ 여기서 AuthenticationManager 직접 호출 ← 이 시점에 인증 시작
            ↓
        AuthenticationManager.authenticate()
            ↓
        인증 성공 → SecurityContext + 세션 저장 → 응답 반환
```

```java
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request, ...) {

    UsernamePasswordAuthenticationToken token =
        new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword());

    // DispatcherServlet을 거쳐 Controller에 도달한 뒤 이 줄에서 인증 시작
    Authentication authentication = authenticationManager.authenticate(token);

    // 이후 SecurityContext와 세션에 저장 (9번 섹션에서 설명)
    ...
}
```

> **정리**: 필터 방식은 Controller 진입 전 FilterChain에서 인증이 끝납니다.
> REST API 방식은 Controller까지 들어온 다음 Controller가 인증을 직접 시작합니다.
> 지금부터는 두 방식 모두에서 `authenticate()` 내부에서 일어나는 일을 봅니다.

## 6. UserDetails & UserDetailsService — 사용자 정보 구조



인증 흐름에서 가장 먼저 알아야 할 것은, Spring Security가 **사용자 정보를 어떤 형태로 다루는가**입니다.

### UserDetails — 사용자 정보 표준 인터페이스

Spring Security는 사용자 정보를 직접 다루지 않습니다.
우리가 만든 `User` 엔티티를 `UserDetails` 인터페이스로 감싸서 전달해야 합니다.

```java
public class CustomUserDetails implements UserDetails {

    private final User user; // 우리 엔티티

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // 권한 목록 반환 (ex: ROLE_USER, ROLE_ADMIN)
        return List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole()));
    }

    @Override
    public String getPassword() { return user.getPassword(); }

    @Override
    public String getUsername() { return user.getEmail(); }

    @Override
    public boolean isAccountNonExpired() { return true; }  // 계정 만료 여부

    @Override
    public boolean isAccountNonLocked() { return true; }   // 계정 잠금 여부

    @Override
    public boolean isCredentialsNonExpired() { return true; } // 비번 만료 여부

    @Override
    public boolean isEnabled() { return user.isActive(); }    // 활성화 여부
}
```

### UserDetailsService — DB 조회 담당

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // Spring Security가 인증할 때 여기를 자동으로 호출함
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("사용자를 찾을 수 없습니다: " + username));

        return new CustomUserDetails(user);
    }
}
```

> `loadUserByUsername()`은 우리가 직접 호출하지 않습니다.
> 인증 흐름 중에 Spring Security가 자동으로 호출합니다.

## 7. 비밀번호 암호화 — PasswordEncoder

**절대 비밀번호를 평문으로 저장하면 안 됩니다.**
실무에서는 항상 `BCryptPasswordEncoder`를 사용합니다.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(); // 단방향 해시, 레인보우 테이블 공격 방어
}
```

**사용 방법**

```java
// 회원가입 시 — 저장 전에 암호화
String encoded = passwordEncoder.encode(rawPassword);
user.setPassword(encoded);
userRepository.save(user);

// 로그인 시 — 직접 비교하지 말고 matches() 사용
boolean isMatch = passwordEncoder.matches(rawPassword, encodedPassword);
```

> **왜 BCrypt인가?**: 같은 비밀번호를 encode해도 매번 다른 결과가 나옵니다(Salt 자동 포함).
> 따라서 DB가 유출되어도 원래 비밀번호를 알아내기 매우 어렵습니다.

## 8. AuthenticationManager & Provider — 인증 처리 중심

UserDetails로 사용자 정보 구조를 알고, PasswordEncoder로 비밀번호 검증 방식을 알았으니
이제 **인증을 실제로 처리하는 중심축**을 봅니다.

```
AuthenticationManager (인터페이스) ← "인증해줘" 요청을 받는 창구
    └─ ProviderManager (구현체)
            ├─ DaoAuthenticationProvider  ← DB 기반 인증 (가장 많이 씀)
            │       ├─ UserDetailsService.loadUserByUsername() 호출
            │       └─ PasswordEncoder.matches() 로 비번 대조
            ├─ OAuth2LoginAuthenticationProvider  ← 소셜 로그인
            └─ (커스텀 Provider 추가 가능)
```

**AuthenticationManager 빈 등록 (직접 주입할 때)**

```java
@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration authConfig) throws Exception {
    return authConfig.getAuthenticationManager();
}
```

**DaoAuthenticationProvider 직접 설정 (PasswordEncoder 연결)**

```java
@Bean
public DaoAuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(customUserDetailsService);
    provider.setPasswordEncoder(passwordEncoder()); // 비번 검증 방식 연결
    return provider;
}
```

## 9. SecurityContext — 인증 성공 후 정보 보관함

`AuthenticationManager.authenticate()`가 성공하면 `Authentication` 객체가 반환됩니다.
이걸 어딘가에 보관해야 이후 요청에서 "이 사람이 누구인지" 알 수 있습니다.
Spring Security는 `SecurityContextHolder`라는 공간을 사용합니다.

```
SecurityContextHolder
    └─ SecurityContext
            └─ Authentication       ← 인증 성공 후 여기에 저장됨
                    ├─ Principal (= UserDetails, 사용자 정보)
                    ├─ Credentials (= 비밀번호, 보통 인증 후 삭제)
                    └─ Authorities (= 권한 목록, ex: ROLE_USER, ROLE_ADMIN)
```

**현재 로그인한 사용자 정보 가져오기**

```java
// Controller나 Service 어디서든 꺼낼 수 있음
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
UserDetails user = (UserDetails) auth.getPrincipal();
String username = user.getUsername();
```

> **실무 포인트**: SecurityContextHolder는 기본적으로 **ThreadLocal**을 사용합니다.
> 요청 스레드마다 독립된 공간을 가집니다. 덕분에 동시에 수천 명이 로그인해도
> 서로의 인증 정보가 섞이지 않습니다.

**인증 후 SecurityContext를 세션에 저장하는 코드**

```java
// 로그인 성공 시, 세션에 인증 컨텍스트 명시적 저장
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);

// HTTP 세션에도 동기화 (이걸 빠뜨리면 다음 요청에서 로그아웃됨!)
securityContextRepository.saveContext(context, request, response);
```

> **자주 하는 실수**: `SecurityContextHolder`에만 저장하고 세션에 저장 안 하는 경우.
> 같은 요청 안에서는 인증이 유지되지만, 다음 요청이 오면 인증이 사라집니다.

## 10. 세션 인증과 토큰 인증

지금까지 배운 모든 컴포넌트를 연결해서 전체 그림을 봅니다.

```
[최초 로그인]
    ↓
1. 클라이언트: POST /login { id, password }
    ↓
2. Controller: AuthenticationManager.authenticate() 호출
    ↓
3. DaoAuthenticationProvider:
      UserDetailsService.loadUserByUsername() → DB에서 사용자 조회
      PasswordEncoder.matches() → 비밀번호 대조
    ↓
4. 인증 성공 → Authentication 객체 반환
    ↓
5. SecurityContextHolder에 Authentication 저장
   + HTTP 세션에도 SecurityContext 명시적 저장
    ↓
6. 서버: 세션 ID(JSESSIONID)를 쿠키에 담아 응답

[이후 모든 요청]
    ↓
7. 브라우저: 쿠키(JSESSIONID)를 자동으로 요청에 포함
    ↓
8. SecurityContextPersistenceFilter: 세션 ID → SecurityContext 복원
    ↓
9. 이후 필터/인터셉터가 인증된 사용자로 처리
```

**쿠키는 어디에 저장되나?**

```
클라이언트(브라우저) 쪽에 저장됩니다.
→ 서버는 세션 ID만 발급하고, 그 ID에 매핑되는 실제 데이터는 서버 메모리에 보관
→ 브라우저는 요청마다 쿠키를 자동 첨부 → 서버가 세션 ID로 사용자를 식별
```

**세션 인증 구현 핵심 포인트 (React + Spring)**

```java
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request,
                                HttpServletRequest req,
                                HttpServletResponse res) {

    // 인증 시도 (Provider → UserDetailsService 호출됨)
    UsernamePasswordAuthenticationToken token =
        new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword());

    Authentication authentication = authenticationManager.authenticate(token);

    // SecurityContextHolder에 저장
    SecurityContext context = SecurityContextHolder.createEmptyContext();
    context.setAuthentication(authentication);
    SecurityContextHolder.setContext(context);

    // 세션에도 명시적으로 저장 (이 단계를 빠뜨리면 다음 요청 시 인증 사라짐)
    securityContextRepository.saveContext(context, req, res);

    return ResponseEntity.ok("로그인 성공");
}
```


## 11. 역할(Role) 기반 인가

인증이 완료된 후, 이제 "이 사람이 이걸 해도 되는가"를 판단합니다.

### URL 단위 인가 — SecurityFilterChain에서 설정

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/").permitAll()                     // 모두 허용
    .requestMatchers("/api/admin/**").hasRole("ADMIN")    // ADMIN만
    .requestMatchers("/api/user/**").hasRole("USER")      // USER만
    .requestMatchers("/api/**").authenticated()           // 로그인만 하면 OK
    .anyRequest().permitAll()
);
```

### 메서드 단위 인가 — @PreAuthorize (실무에서 자주 씀)

```java
@Configuration
@EnableMethodSecurity  // 이 어노테이션 필수!
public class SecurityConfig { ... }
```

```java
@RestController
public class AdminController {

    @GetMapping("/admin/users")
    @PreAuthorize("hasRole('ADMIN')")  // ADMIN 역할이 없으면 403
    public List<User> getAllUsers() { ... }

    @DeleteMapping("/posts/{id}")
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public void deletePost(@PathVariable Long id) { ... }
}
```

**Spring Security에서 Role 명명 규칙**

```
DB에 저장할 때:   "ADMIN"    또는   "ROLE_ADMIN"
코드에서 체크:    hasRole("ADMIN")       →  내부적으로 "ROLE_ADMIN"과 비교
                  hasAuthority("ROLE_ADMIN")  →  prefix 없이 그대로 비교
```

## 12. 다중 서버 환경에서의 세션 확장 전략

### 문제 상황

```
사용자가 서버 A에서 로그인 → 세션이 서버 A 메모리에 저장
다음 요청이 서버 B로 라우팅됨 → 서버 B에는 세션 없음 → 로그아웃됨!
```

### 해결책: 세션 스토리지 외부화 (Redis)

```
클라이언트
    ↓
[로드 밸런서]
    ├── 서버 A ──┐
    ├── 서버 B ──┼──→ [Redis] ← 모든 서버가 공유하는 세션 저장소
    └── 서버 C ──┘
```

**Spring Session + Redis 설정**

```gradle
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
implementation 'org.springframework.session:spring-session-data-redis'
```

```java
@Configuration
@EnableRedisHttpSession
public class SessionConfig {
    // 이것만 추가하면 세션이 자동으로 Redis에 저장됨
}
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  session:
    store-type: redis
    timeout: 30m  # 세션 유효 시간
```

> **실무 포인트**: 대부분의 서비스는 Redis를 세션 스토리지로 사용합니다.
> 속도가 빠르고, TTL(만료 시간) 설정이 쉽고, 여러 서버가 공유할 수 있습니다.

## 13. CORS & CSRF 설정

### CORS — 프론트(React)와 백(Spring)이 분리된 경우 필수

4번 섹션에서 CORS 개념을 설명했습니다. Spring Security에서는 아래와 같이 설정합니다.

```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));  // React 개발 서버
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);  // 쿠키 포함 요청 허용 (세션 인증 필수)

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

### CSRF — REST API라면 보통 비활성화

```java
// 세션 + 브라우저 기반: CSRF 활성화 (기본값)
// REST API + JWT 방식: CSRF 비활성화 가능
http.csrf(csrf -> csrf.disable());
```

**CSRF가 뭔지 간단히:**
```
악의적인 사이트 malicious.com에서 우리 서버로 요청을 보내는 공격
→ 브라우저가 쿠키를 자동으로 첨부하기 때문에 발생
→ CSRF 토큰으로 방어: "진짜 우리 사이트에서 보낸 요청에만 토큰을 붙임"
→ REST API + JSON은 Content-Type 자체가 보호막이 되어 CSRF 위험이 낮음
```

## 14. 실무에서 자주 마주치는 패턴

### 현재 로그인 사용자 편하게 가져오기

```java
// @AuthenticationPrincipal 어노테이션 활용
@GetMapping("/my/profile")
public ResponseEntity<UserDto> getMyProfile(
        @AuthenticationPrincipal CustomUserDetails userDetails) {
    // SecurityContextHolder 직접 안 써도 됨
    User user = userDetails.getUser();
    return ResponseEntity.ok(UserDto.from(user));
}
```

### 인증 실패 / 인가 실패 커스텀 응답

```java
http
    .exceptionHandling(ex -> ex
        // 인증 안 됨 (로그인 안 한 상태) → 401
        .authenticationEntryPoint((request, response, authException) -> {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"로그인이 필요합니다\"}");
        })
        // 권한 없음 (로그인은 했지만 접근 불가) → 403
        .accessDeniedHandler((request, response, accessDeniedException) -> {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("{\"error\": \"접근 권한이 없습니다\"}");
        })
    );
```

### 로그아웃 처리

```java
http.logout(logout -> logout
    .logoutUrl("/api/logout")
    .logoutSuccessHandler((request, response, authentication) -> {
        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write("{\"message\": \"로그아웃 완료\"}");
    })
    .invalidateHttpSession(true)      // 세션 무효화
    .deleteCookies("JSESSIONID")      // 쿠키 삭제
    .clearAuthentication(true)        // SecurityContext 초기화
);
```

## 전체 인증 흐름 요약

```
1. 클라이언트  →  POST /login { username, password }
                        ↓
2. Controller  →  AuthenticationManager.authenticate()
                        ↓
3. ProviderManager  →  DaoAuthenticationProvider
                        ↓
4. UserDetailsService  →  DB 조회 (loadUserByUsername)
                        ↓
5. PasswordEncoder.matches() 로 비밀번호 검증
                        ↓
6. 성공 시 Authentication 객체 반환
                        ↓
7. SecurityContextHolder + 세션에 저장
                        ↓
8. 이후 요청마다 세션 ID(쿠키)로 SecurityContext 복원
                        ↓
9. SecurityContext의 Authentication으로 인가 처리
```

> **마지막 당부**: Spring Security는 처음에 설정이 많아서 어렵게 느껴집니다.
> 하지만 "요청이 오면 필터가 먼저 잡는다", "인증 정보는 SecurityContext에 있다",
> "권한 체크는 그 다음이다" — 이 세 가지 흐름만 머릿속에 새겨두면
> 나머지 설정들은 필요할 때 찾아서 쓸 수 있습니다.
> 지금 당장 모든 걸 외우려 하지 말고, 흐름을 이해하는 데 집중하세요.
