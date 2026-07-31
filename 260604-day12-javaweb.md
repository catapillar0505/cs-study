## **웹서버**

웹서버는 정적인 파일을 클라이언트에게 전달하는 서버야. HTML, CSS, 이미지, JS 파일 같은 걸 그냥 있는 그대로 돌려줘.

예: Nginx, Apache



<웹서버의 역할>
(1) 라우팅

(2) 로드밸런싱



**라우팅이란?**

(WAS로 보낼지 말지 결정)

Nginx

    ↓ 정적 요청 → Nginx가 직접 응답 (HTML, 이미지)

    ↓ 동적 요청 → WAS로 전달





**로드밸런싱이란?**

WAS 서버가 여러 대 있을 때, 요청을 골고루 분산시켜주는 것



사용자 요청 100개

        ↓

    Nginx (로드밸런서)

    ↙    ↓    ↘

WAS1  WAS2  WAS3

(33개) (33개) (34개)

WAS 한 대에 요청이 몰리면 터지니까, Nginx가 나눠주는 거야.

## 

## **WAS (Web Application Server)**



동적인 웹 콘텐츠(사용자 요청에 따라 달라지는)를 처리하여 로직을 실행시켜주는 서버



예: Tomcat, Node.js, Django, Spring Boot

자바에서는 서블릿 컨테이너 역할



Spring = 참고로 요즘 Spring을 쓰면 서블릿을 직접 짤 일은 거의 없어. Spring이 내부적으로 DispatcherServlet 하나로 모든 요청을 받아서 Controller에게 나눠주거든



실무에서 Spring Boot를 톰캣에 올려서 쓰면, 동적 처리 + DB 연동 다 되니까 기능적으로는 WAS 역할을 충분히 해. 그래서 넓은 의미로 WAS라고 부르는 거야.



#### WAS 실제 동작 흐름

사용자 요청

    ↓

웹서버 (Nginx)

    ↓ 정적 파일이면 → 바로 응답 (HTML, 이미지 등)

    ↓ 동적 요청이면 → WAS로 전달

WAS

(대표적인 WAS: Apache Tomcat)

  ↓

어떤 로직에게 넘길지 결정 (라우팅)

    ↓

DB 조회 / 비즈니스 로직 실행 (서블릿, View, 미들웨어 등)

    ↓

결과를 웹서버 통해 응답



## **순수 서블릿 (톰캣)**

Tomcat = 서블릿을 실행시켜주는 컨테이너 역할



**서블릿이란?**

WAS 안에서 http 요청과 응답에 대해 처리하는 Java 클래스





###### **(1) 서블릿 컨테이너(WAS)의 라이프사이클 관리**

**= 톰캣이 서블릿을 싱글톤으로 1개만 만들어서 모든 요청에 재사용**



개발자는 만들고 없애는 걸 신경 안 써도 돼.

개발자는 그냥 클래스만 만들어 놓음

톰캣이 알아서 생성, 실행, 소멸(라이프 사이클)까지 관리

#### 



###### **(2) 저수준 HTTP 제어**

**= HttpServlet은 HTTP 관련 귀찮은 것들을 다 구현해놓은 클래스고, 개발자는 그걸 상속받아서 doGet, doPost만 오버라이드하면 돼.**



저수준이란?

HttpServlet이 추상화해서 감춰놓은 것들을 직접 다루는 것.



개발자가 보통 다루는 것  →  doGet(), doPost(), getParameter()

저수준 HTTP 제어        →  소켓, 바이트 스트림, HTTP 패킷 직접 파싱, 파라미터 직접 추출



Java I/O

    ↑

Servlet (인터페이스)

    ↑

GenericServlet (추상 클래스)

    ↑

HttpServlet (추상 클래스)  ← 우리가 상속받는 것

    ↑

LoginServlet (개발자가 만드는 것)



핵심: service() 메서드가 HTTP 메서드("GET","POST" ..)를 분기해줌

개발자는 service()를 직접 건드릴 필요 없이 필요한 메서드만 오버라이드하면 돼.



\- WAS가 웹 브라우저로부터 서블릿 요청을 받으면 `HttpServletRequest` 객체가 생성됨

\- 요청 헤더, 요청 파라미터, 쿠키 등의 정보를 처리할 수 있는 기능



 HttpServletResponse 클래스

응답을 위한 출력 스트림 추출, 응답 주소 처리, 응답 데이터 타입 및 문자셋 설정 등의 기능 포함



###### **(3)URL Mapping**

\- 작성된 서블릿은 브라우저를 통해 호출 가능

\- 서블릿을 호출하기 위해 지정한 이름을  URL Mapping 값이라고 함



**순수 서블릿 방식의 문제 => URL마다 서블릿을 일일이 만들어야 함**



## **DispatcherServlet**

**DispatcherServlet →  모든 URL 받아서 Controller에게 분배**

서블릿 대신 Controller만 만들면 됨

모든 요청의 관문 역할을 해서 Front Controller 패턴



모든 요청을 DispatcherServlet 하나가 받음

    ↓

어느 Controller로 보낼지 결정 (Handler Mapping)

    ↓

해당 Controller 실행





## **프레임워크**

**미리 만들어 놓은 클래스나 인터페이스 등을 제공**



**Spring**

Java 애플리케이션 개발을 편하게 해주는 프레임워크.

DI(의존성 주입), AOP 등 개발 편의 기능 제공.

근데 혼자 실행은 못 해 — 톰캣이 있어야 돌아가.

톰캣을 외부에 별도 설치하고, 거기에 Spring 앱을 올려야 해.

war 파일로 빌드해서 톰캣에 배포



**Spring Boot**

Spring을 더 편하게 쓰게 해주는 프레임워크.

톰캣을 내장하고 있어서 별도 설치 불필요.



Spring MVC   →  DispatcherServlet를 개발자가 직접 web.xml에 등록

Spring Boot  →  @EnableAutoConfiguration이 자동으로 등록





## **서블릿 컨테이너 vs 스프링 컨테이너**

톰캣 (서블릿 컨테이너)

    └── DispatcherServlet

            └── 스프링 컨테이너

                    ├── LoginController

                    ├── LoginService

                    └── UserRepository

<톰캣이 하는일>

HTTP 요청 수신

서블릿 생명주기 (init, service, destroy)

DispatcherServlet 생성 및 관리



<스프링 컨테이너가 관리하는 것>

개발자가 만든 객체들 (Bean) 관리

@Controller, @Service, @Repository, @Component

의존성 주입 (DI)



## **리플렉션**

실행 중(런타임)에 클래스 정보를 동적으로 분석하고 조작하는 것.

**기본 생성자 필요**



**리플렉션 쓰는 곳**

&nbsp;   \*\*├── Jackson   →  JSON ↔ 객체 변환\*\*

    \*\*├── 스프링    →  Bean 생성, 의존성 주입\*\*

    \*\*└── JPA      →  엔티티 객체 생성\*\*






## **프론트 렌더링 방식**

CSR: 브라우저가 HTML 만듦 (예시: React, Vue, Angular)

SSR: 서버가 HTML 만듦 (예시: JSP, Thymeleaf (Spring), Django 템플릿)

요즘은 SSR + CSR 혼합을 많이 써.

첫 페이지 로딩 → SSR (빠르게 보여주기 + SEO)

이후 인터랙션  → CSR (부드러운 사용자 경험)



React+Vite는 순수 CSR, Next.js는 React 기반 프레임워크로 SSR·CSR을 모두 지원해



순위프론트특징

1위React (CSR)가장 범용적, 생태계 압도적

2위Vue.js (CSR)국내 특히 많음, 러닝커브 낮음

3위Angular (CSR)대기업/금융권에서 많이 씀

4위Next.js (SSR+CSR) SEO 필요한 서비스에서 증가 중

검색엔진 최적화 — 구글, 네이버 같은 검색엔진에서 내 사이트가 상위에 노출되게 만드는 것.

CSR이 SEO에 불리한 이유 크롤러가 CSR 페이지를 읽으면 JS를 실행 안 하거나 늦게 하는 크롤러는 내용을 파악 못 함.



## **gradle vs maven**

둘 다 Java 프로젝트 빌드 도구야. 쉽게 말하면:

"의존성 관리 + 컴파일 + 테스트 + 패키징"을 자동으로 해주는 것

Maven은 XML 기반의 전통적인 빌드 도구, Gradle은 더 빠르고 간결한 최신 빌드 도구. 신규 프로젝트면 Gradle 써.

mavenCentral() - maven으로부터 받아옴



jar = java archive 파일 = 내 코드를 패키징한 것

의존성(라이브러리)들도 각각 JAR 파일로 존재해.

Spring Boot JAR는 특별해

Spring Boot는 Fat JAR (Uber JAR) 방식으로 빌드해서, 의존성까지 전부 하나에 담아:



## Redirect vs Forward 사용 이유

### 핵심 개념 먼저

**Forward** → 서버 내부에서 요청을 넘김. 브라우저는 URL이 바뀐 줄 모름.
**Redirect** → 서버가 브라우저한테 "이 URL로 다시 요청해"라고 응답. 브라우저 URL이 바뀜.

---

### DB 변경(INSERT/UPDATE/DELETE) → Redirect 써야 하는 이유

**문제 상황:**
```
브라우저 → POST /board/write (글 작성) → 서버가 INSERT 실행
→ Forward로 결과 페이지 보여줌
→ 브라우저 URL은 여전히 POST /board/write
→ 사용자가 F5(새로고침) 누름
→ 브라우저: "이전 POST 요청 다시 보낼까요?"
→ 사용자가 확인 누름
→ INSERT 또 실행됨 → 중복 데이터 생성!
```

이게 바로 **"새로고침 중복 제출 문제"**다.

**Redirect로 해결:**
```
POST /board/write → INSERT 실행
→ Redirect → GET /board/list
→ 브라우저 URL이 GET /board/list로 바뀜
→ F5 눌러도 GET /board/list 재요청 → 그냥 목록 조회만 됨
```

이 패턴을 **PRG 패턴 (Post → Redirect → Get)** 이라고 부른다.

---

### DB 조회(SELECT) → Forward 써도 되는 이유

조회는 몇 번을 반복해도 데이터가 변하지 않음. F5로 같은 요청이 재실행돼도 부작용이 없다.

그리고 Forward는 서버 내부에서 처리하기 때문에 네트워크 왕복이 한 번 줄어들어서 약간 더 빠르다.

---

### 한 줄 정리

| 상황 | 방식 | 이유 |
|---|---|---|
| INSERT/UPDATE/DELETE | Redirect | 새로고침 시 중복 실행 방지 (PRG 패턴) |
| SELECT | Forward | 중복돼도 무해하고, 서버 왕복 1번 절약 |

**핵심은 "멱등성"** 이다. 같은 요청을 여러 번 해도 결과가 같으면(SELECT) 포워드, 중복 실행하면 안 되면(INSERT 등) 리다이렉트.

## **http 상태코드**

200  →  정상

201  →  생성 성공

204  →  삭제 성공

301 -> 우리 이사했어요

302 -> 임시 공간

400  →  내가 요청 잘못 보냄

401  →  로그인 필요 (너누구야)

403  →  권한 없음 (권한 없어)

404  →  없는 URL

500  →  서버 터짐

502  →  Nginx-WAS 연결 문제

503  →  서버 죽음



---



401  →  "너 누구야?" (신원 확인 안 됨, 로그인 필요)

403  →  "너 누군지 알아, 근데 안 돼" (권한 없음)



---



301  →  도메인 변경 (old.com → new.com)

         HTTP → HTTPS 강제 이동

 

302  →  로그인 후 메인으로 이동

         A/B 테스트로 임시 다른 페이지 보여줄 때

---



사용자가 브라우저에 http://old.com 입력

    ↓

서버: "301 - 우리 https://new.com 으로 영구 이동했어"

    ↓

브라우저 자동으로 https://new.com 으로 이동

    ↓

다음부터 브라우저가 old.com 요청 안 하고

바로 new.com으로 감 (캐싱)



---

## **Stateless 기술**

**=> 쿠키, JWT**

로그인 성공

    ↓

서버: JWT 토큰 발급 (서버엔 저장 안 함)

    ↓

브라우저: 로컬스토리지에 JWT 저장

    ↓

다음 요청마다 헤더에 JWT 담아서 전송

Authorization: Bearer eyJhbGci...

    ↓

서버: 서명 검증 → "jina구나" (DB 조회 없이)





## **Stateful 기술**



HTTP Stateless 문제 (서버가 클라이언트를 기억 못 해. 장바구니 보여줘 - 너누구야?)

    ↓

쿠키 등장 (브라우저가 기억)

    ↓

쿠키 보안 문제 (변조 가능)

    ↓

세션 등장 (서버가 기억, 브라우저엔 ID만)



1. 서버: 메모리(또는 DB)에 세션 저장소를 만들고

고유한 Session ID를 발급한 뒤, 이를 쿠키에 담아 브라우저에 보냄 (`Set-Cookie: JSESSIONID=abc123`)



2. 클라이언트: 다음 요청부터 이 Session ID가 담긴 쿠키를 서버에 제출함



1. 서버: 제출된 Session ID로 서버 내 세션 저장소를 조회하여 저장된 정보를 꺼내서 사용
