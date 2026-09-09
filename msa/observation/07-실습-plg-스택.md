> [← observation 목차](README.md)

# 실습 — PLG 스택 구축

## 한 줄로

**Prometheus(지표) + Zipkin(추적) + Loki(로그) + Grafana(화면)** 를 Docker로 띄우고, Spring Boot 하나를 물려 **한 요청이 세 화면에 어떻게 나타나는지** 눈으로 확인한다.

## 무엇을 만드나

| 관측 요소 | 담당 도구 | 포트 | 역할 |
|---|---|---|---|
| **Metrics** | Prometheus | `9090` | 시계열 수치 수집·저장 |
| **Tracing** | Zipkin | `9411` | Trace/Span 수집·타임라인 조회 |
| **Logging** | Grafana Loki | `3100` | 로그 저장 (라벨 기반) |
| **Visualization** | Grafana | `3000` | 셋을 **한 화면에서** 조회 |

> **PLG = Prometheus + Loki + Grafana.** ELK에 대응하는 Grafana 진영의 조합 이름입니다.
> 여기에 추적으로 Zipkin을 얹었습니다. (Grafana 진영의 추적 도구는 Tempo지만, Zipkin이 가볍고 UI가 직관적입니다.)

```
    ┌──────────────────────────────────────────────────┐
    │              SPRING BOOT APPLICATION             │
    │              (observability-service :8080)       │
    └────────┬───────────────┬────────────────┬────────┘
             │               │                │
 Metrics Pull │         Traces Push           │ Logs Push
   /actuator/ │        (Zipkin HTTP)          │ (Loki HTTP)
   prometheus │               │                │
    ┌────────▼───────┬───────▼────────┬───────▼────────┐
    │   Prometheus   │     Zipkin     │  Grafana Loki  │
    │     :9090      │     :9411      │     :3100      │
    └────────┬───────┴───────┬────────┴───────┬────────┘
             └───────────────┼────────────────┘
                             │  통합 조회
                  ┌──────────▼─────────┐
                  │  Grafana Dashboard │
                  │       :3000        │
                  └────────────────────┘
```

> **화살표 방향을 눈여겨보세요.**
> **지표만 Pull(당김)** 이고, **추적과 로그는 Push(밀기)** 입니다.
> 지표는 "지금 값이 얼마냐"라 언제 물어봐도 되지만, **추적·로그는 사건이 일어난 그 순간에만 존재**하므로
> 애플리케이션이 직접 보내야 합니다. → [03의 Pull vs Push](03-prometheus와-grafana.md)

## 프로젝트 구조

```
/observe-project
├── docker
│   ├── prometheus/prometheus.yml                    # 수집 대상 설정
│   ├── loki/loki-config.yml                         # Loki 엔진 설정
│   ├── grafana/provisioning/datasources/
│   │   └── datasources.yml                          # Grafana 데이터소스 자동 등록
│   └── docker-compose.yml                           # 인프라 4종 통합 실행
│
└── observability-service                            # Spring Boot 애플리케이션
    ├── build.gradle
    └── src/main/
        ├── java/com/example/observe/
        │   ├── ObservabilityServiceApplication.java
        │   ├── config/ObserveConfig.java
        │   └── controller/ObserveController.java
        └── resources/
            ├── application.yml
            └── logback-spring.xml                   # Loki 전송 설정
```

---

## Step 1. 인프라 (Docker Compose)

### 기초 — Docker Compose의 두 가지 규칙

이 실습에서 헷갈리는 지점이 전부 여기서 나옵니다.

**① 컨테이너끼리는 "서비스 이름"으로 통신한다**

```yaml
services:
  prometheus:   # ← 이 이름이 그대로 호스트명이 된다
  grafana:
```

Grafana 컨테이너 안에서 Prometheus를 부를 때 주소는 **`http://prometheus:9090`** 입니다.
`localhost`가 **아닙니다.** 컨테이너 안에서 `localhost`는 **자기 자신**을 뜻하니까요.

**② 컨테이너에서 "내 PC"를 부르려면 특별한 이름이 필요하다**

Spring Boot는 컨테이너가 아니라 **내 PC에서** 도는데, Prometheus는 컨테이너 안에 있습니다.

```
   Prometheus 컨테이너에서 localhost:8080  → 컨테이너 자기 자신 (아무것도 없음) ❌
   Prometheus 컨테이너에서 host.docker.internal:8080 → 내 PC ✅
```

**`host.docker.internal`** 이 **"이 컨테이너를 띄운 호스트 PC"** 를 가리키는 특별한 이름입니다.

> ⚠️ **WSL을 쓴다면 주의하세요.**
> Docker Desktop에서 `host.docker.internal`은 **Windows 호스트**를 가리킵니다.
> Spring Boot를 **WSL 안에서** 실행 중이라면 찾지 못합니다.
> 이 경우 WSL 터미널에서 `ip addr show eth0` 로 IP를 확인해 직접 적어야 합니다.

### `docker/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 5s          # 얼마나 자주 긁어올 것인가 (실습용으로 짧게)
  evaluation_interval: 5s      # 알림 규칙을 얼마나 자주 검사할 것인가

scrape_configs:
  - job_name: 'observability-service'      # 이 이름이 라벨 job= 으로 붙는다
    metrics_path: '/actuator/prometheus'   # 기본값은 /metrics 라서 반드시 지정
    static_configs:
      - targets: ['host.docker.internal:8080']
```

| 설정 | 왜 |
|---|---|
| `scrape_interval: 5s` | 실습에서는 그래프가 빨리 움직여야 확인이 쉬움. **운영은 15s~1m** |
| `metrics_path` | Prometheus 기본값이 `/metrics`인데, **Spring은 `/actuator/prometheus`** |
| `job_name` | Grafana에서 `{job="observability-service"}` 로 골라낼 때 쓰는 이름 |

### `docker/loki/loki-config.yml`

```yaml
auth_enabled: false            # 멀티 테넌트 인증 끄기 (로컬 실습용)

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory          # 단일 노드라 메모리로 충분
  replication_factor: 1
  path_prefix: /tmp/loki

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb              # 인덱스 저장 엔진
      object_store: filesystem # 로그 본문 저장 위치 (운영에서는 s3)
      schema: v13
      index:
        prefix: index_
        period: 24h            # 인덱스 파일을 하루 단위로 새로 만듦

storage_config:
  filesystem:
    directory: /tmp/loki/chunks   # 압축된 로그 덩어리가 쌓이는 곳

limits_config:
  allow_structured_metadata: true
  reject_old_samples: true
  reject_old_samples_max_age: 168h   # 1주일보다 오래된 로그는 거부 (디스크 보호)
  ingestion_rate_mb: 10              # 초당 10MB 까지만 수집 (폭주 방지)
  ingestion_burst_size_mb: 20
```

**설정에서 읽어야 할 것**

| 항목 | 의미 |
|---|---|
| `auth_enabled: false` | Loki는 원래 **여러 조직의 로그를 격리**해 담을 수 있음(멀티 테넌시). 실습에선 불필요 |
| **index / chunks 분리** | [06에서 본 Loki의 핵심](06-분산-로깅.md) — **인덱스와 본문을 따로 저장**한다 |
| `reject_old_samples` | 실수로 옛날 로그를 대량 밀어 넣어 **디스크를 채우는 사고 방지** |
| `ingestion_rate_mb` | 무한 루프 로그로 **Loki가 죽는 것 방지** |

> **`filesystem`은 실습용입니다.** 운영에서는 `s3` 같은 오브젝트 스토리지를 씁니다.
> 그게 Loki가 싼 이유이기도 합니다 — **값싼 스토리지에 압축해서 던져두면 되니까요.**

### `docker/grafana/provisioning/datasources/datasources.yml`

Grafana에 데이터 소스를 **손으로 등록하지 않고 자동으로 붙이는** 설정입니다.

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus                 # 대시보드에서 참조할 고정 ID
    url: http://prometheus:9090     # ★ localhost 아님. 서비스 이름
    access: proxy
    isDefault: true
    editable: false

  - name: Zipkin
    type: zipkin
    uid: zipkin
    url: http://zipkin:9411
    access: proxy
    editable: false

  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
    access: proxy
    editable: false
    jsonData:
      maxLines: 1000                # 한 번에 가져올 최대 로그 줄 수
```

| 항목 | 의미 |
|---|---|
| **`url`이 서비스 이름** | 위 "규칙 ①". Grafana 컨테이너 → Prometheus 컨테이너로 가는 주소 |
| **`access: proxy`** | **Grafana 서버가 대신** 요청을 보냄. `direct`는 브라우저가 직접 보내서 CORS 문제 발생 |
| **`uid` 고정** | 대시보드 JSON이 이 ID로 데이터소스를 참조. 고정해두면 이식이 쉬움 |
| **`editable: false`** | UI에서 못 고치게 잠금 — **"설정 파일로 관리한다"는 선언** |

> **프로비저닝(Provisioning)** 은 **"시작할 때 설정을 자동으로 밀어 넣는 것"** 입니다.
> 이 파일 덕분에 Grafana에 로그인하자마자 **아무 설정 없이 바로 조회**할 수 있습니다.

### `docker/docker-compose.yml`

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - "9090:9090"
    extra_hosts:
      - "host.docker.internal:host-gateway"    # ★ 리눅스에서 호스트 PC 를 찾기 위해

  zipkin:
    image: openzipkin/zipkin:3.0.6
    container_name: zipkin
    ports:
      - "9411:9411"

  loki:
    image: grafana/loki:3.0.0
    container_name: loki
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yml
    command: -config.file=/etc/loki/local-config.yml
    ports:
      - "3100:3100"

  grafana:
    image: grafana/grafana:10.4.0
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
      - zipkin
      - loki
```

| 설정 | 의미 |
|---|---|
| **`volumes`** | 내 PC의 설정 파일을 **컨테이너 안에 끼워 넣기**. 고치고 재시작만 하면 반영됨 |
| **`ports: "9090:9090"`** | `내PC포트:컨테이너포트`. 앞이 내가 브라우저로 접속할 포트 |
| **`extra_hosts`** | 리눅스 도커에는 `host.docker.internal`이 기본으로 없어서 **직접 만들어 주는 것** |
| **`depends_on`** | 시작 **순서**만 보장. "준비 완료"까지 기다리진 않음 |

> **`image` 태그를 버전으로 고정한 것**을 눈여겨보세요. `latest`를 쓰면
> 어느 날 갑자기 버전이 올라가 **설정 형식이 안 맞아 안 뜨는** 일이 생깁니다.

---

## Step 2. 애플리케이션

### 의존성 — 세 갈래

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-webmvc'
  implementation 'org.springframework.boot:spring-boot-starter-actuator'   // ① 관측의 출발점

  // @Timed 를 동작시키려면 AOP 가 필요
  implementation 'org.springframework:spring-aop'
  implementation 'org.aspectj:aspectjweaver'

  // 1. Metrics — Prometheus 포맷으로 내보내기
  implementation 'io.micrometer:micrometer-registry-prometheus'

  // 2. Tracing — OTel 브릿지 + Zipkin 으로 내보내기
  implementation 'io.micrometer:micrometer-tracing-bridge-otel'
  implementation 'io.opentelemetry:opentelemetry-exporter-zipkin'

  // 3. Logging — Loki 로 직접 전송하는 Logback 애펜더
  implementation 'com.github.loki4j:loki-logback-appender:1.5.1'
}
```

**세 줄이 [01의 3대 요소](01-관측-가능성이란.md)와 정확히 대응합니다.**

| 의존성 | 하는 일 |
|---|---|
| `micrometer-registry-prometheus` | 메트릭을 **Prometheus가 읽을 텍스트 형식**으로 변환 |
| `micrometer-tracing-bridge-otel` | Micrometer API를 **OTel 구현체에 연결** → [05의 브릿지](05-opentelemetry.md) |
| `opentelemetry-exporter-zipkin` | 수집한 Span을 **Zipkin 형식으로 전송** |
| `loki-logback-appender` | Logback 로그를 **Loki로 HTTP 전송** |

> **추적이 의존성 2개인 이유**: `bridge-otel`은 **계측**을, `exporter-zipkin`은 **전송**을 담당합니다.
> Jaeger로 바꾸고 싶으면 **exporter만** 갈아끼우면 됩니다. → [05](05-opentelemetry.md)의 표준화 이야기.

> ⚠️ `spring-boot-starter-aop` 대신 하위 의존성 둘을 직접 넣은 것은 **일시적인 우회**입니다.
> 스타터가 정상화되면 그걸 쓰면 됩니다.

### `application.yml`

```yaml
spring:
  application:
    name: observability-service     # 메트릭 태그·로그 라벨·Zipkin 서비스명에 전부 쓰임

management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus    # ① 안 열면 404
  endpoint:
    health:
      show-details: always
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}   # 모든 메트릭에 공통 태그 부착

  tracing:
    sampling:
      probability: 1.0              # ② 기본 0.1(10%) → 실습은 전부 수집

  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans   # ③ Span 을 보낼 곳

loki:
  url: http://localhost:3100/loki/api/v1/push        # ④ 로그를 보낼 곳
```

**실습에서 반드시 챙겨야 할 네 줄**

| # | 안 하면 |
|---|---|
| ① `exposure.include` | Prometheus가 `/actuator/prometheus`를 못 읽어 **404** → [02](02-메트릭.md) |
| ② `probability: 1.0` | **10번 중 9번은 추적이 수집 안 됨** → "요청했는데 Zipkin에 없다" → [04](04-분산-추적.md) |
| ③ Zipkin 엔드포인트 | Span이 어디로도 안 감 |
| ④ Loki URL | 로그가 콘솔에만 찍히고 중앙에 안 모임 |

> **`spring.application.name` 하나가 세 곳에서 쓰입니다.**
> 메트릭 태그(`application=`), 로그 라벨(`app=`), Zipkin 서비스 이름.
> **이름이 다르면 세 화면에서 같은 서비스인 줄 모릅니다.**

> ⚠️ **여기 주소는 `localhost`가 맞습니다.** Spring Boot는 **컨테이너 밖(내 PC)** 에서 돌기 때문입니다.
> 컨테이너 안에서 부를 때만 서비스 이름을 씁니다. **어디서 부르느냐에 따라 주소가 달라집니다.**

### `logback-spring.xml`

로그를 **두 곳으로 동시에** 내보냅니다 — 콘솔(사람용)과 Loki(중앙 저장용).

```xml
<configuration>
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <springProperty scope="context" name="appName" source="spring.application.name"/>
  <springProperty scope="context" name="lokiUrl" source="loki.url"/>

  <!-- ① 콘솔: 사람이 읽는 형식 (MDC 의 traceId, spanId 포함) -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{traceId:-},%X{spanId:-}] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <!-- ② Loki: JSON 구조화 로그 -->
  <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
    <http>
      <url>${lokiUrl}</url>
    </http>
    <format>
      <label>
        <pattern>app=${appName},host=${HOSTNAME},level=%level</pattern>
      </label>
      <message>
        <pattern>{"traceId":"%X{traceId:-}","spanId":"%X{spanId:-}","logger":"%logger{36}","message":"%replace(%msg){'[\r\n]+', ' '}","exception":"%replace(%ex){'[\r\n]+', ' '}"}</pattern>
      </message>
    </format>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE" />
    <appender-ref ref="LOKI" />
  </root>
</configuration>
```

**여기서 읽어야 할 세 가지**

**① `%X{traceId:-}` — MDC에서 값 꺼내기**

`%X{키}` 가 **MDC에서 값을 꺼내는 문법**입니다. → [06의 MDC](06-분산-로깅.md)
`:-` 는 **"값이 없으면 빈 문자열"** 이라는 기본값 표시입니다. (없을 때 `traceId_IS_UNDEFINED` 같은 게 찍히는 걸 막습니다.)

**② `label` vs `message` — Loki 설계가 그대로 드러납니다**

```xml
<label>   app=..., host=..., level=...   </label>   ← 색인되는 것 (적게!)
<message> {"traceId":"...", "message":"..."} </message>   ← 압축 저장되는 본문
```

**[06에서 본 "라벨만 색인"이 여기 그대로 나타납니다.**
`level`은 라벨이라 `{level="ERROR"}` 로 빠르게 걸러지고, `traceId`는 본문이라 텍스트로 검색합니다.

> **왜 traceId를 라벨에 안 넣나?** 요청마다 값이 달라서 **라벨 종류가 무한히 늘어나기 때문**입니다.
> → [카디널리티 폭발](02-메트릭.md)

**③ `%replace(%msg){'[\r\n]+', ' '}` — 줄바꿈 제거**

JSON 한 줄 안에 **줄바꿈이 들어가면 형식이 깨집니다.** 스택 트레이스는 줄바꿈투성이라
**공백으로 치환**해서 한 줄로 만듭니다.

### 컨트롤러 — 일부러 느리게, 일부러 실패하게

```java
@Slf4j
@RestController
public class ObserveController {

  @Timed(value = "api.hello.latency", description = "Hello API 응답 지연 시간 및 호출 횟수")
  @GetMapping("/hello")
  public String hello(@RequestParam(defaultValue = "World") String name) throws InterruptedException {
    log.info("Incoming request received for name: {}", name);

    // ① 0~1000ms 무작위 지연 — 추적 타임라인에 편차를 만들기 위해
    int latency = ThreadLocalRandom.current().nextInt(1000);
    Thread.sleep(latency);

    // ② 800ms 초과 시 의도적 500 에러 — ERROR 로그와 Zipkin 에러 태그 확인용
    if (latency > 800) {
      log.error("High latency threshold exceeded! Latency: {}ms", latency);
      throw new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "...");
    }

    log.info("Request processed successfully in {}ms", latency);
    return "Hello, " + name + "! (Latency: " + latency + "ms)";
  }
}
```

**이 짧은 메서드 하나가 세 요소를 전부 만들어냅니다.**

| 요소 | 무엇이 생기나 |
|---|---|
| **Metrics** | `@Timed` → `api_hello_latency_seconds_count / _sum / _max` → [02](02-메트릭.md) |
| **Tracing** | 메서드 진입 시 **Root Span 생성**, traceId/spanId 부여 → [04](04-분산-추적.md) |
| **Logging** | MDC 덕분에 로그에 **traceId 자동 부착** → [06](06-분산-로깅.md) |

> **일부러 지연과 에러를 만드는 것**을 **카오스 엔지니어링**이라고 합니다.
> 실제 장애는 재현하기 어려우니 인위적으로 만들어 **관측 장치가 제대로 작동하는지** 검증합니다.
> → [msa/06](../06-장애-격리.md)에서도 같은 기법이 나옵니다.

`@Timed`가 동작하려면 이를 처리할 빈이 필요합니다.

```java
@Configuration
public class ObserveConfig {
  @Bean
  public TimedAspect timedAspect(MeterRegistry registry) {
    return new TimedAspect(registry);
  }
}
```

---

## Step 3. 실행과 검증

### 1) 인프라 기동

```bash
cd docker
docker compose up -d
docker compose ps
```

`prometheus`, `zipkin`, `loki`, `grafana` 넷 다 **Up** 상태여야 합니다.

### 2) 애플리케이션 기동

```bash
cd observability-service
../gradlew bootRun
```

### 3) 수집이 되는지 먼저 확인 — Prometheus Targets

**`http://localhost:9090/targets`**

`observability-service`의 상태가 **`UP`** 이어야 합니다.

> **여기서 막히면 뒤가 전부 안 됩니다.** 먼저 이걸 확인하세요.
>
> | 증상 | 원인 |
> |---|---|
> | `DOWN` + connection refused | 앱이 안 떠 있거나 **포트 불일치** |
> | `DOWN` + 404 | **Actuator 노출 설정 누락** 또는 `metrics_path` 오타 |
> | 아예 목록에 없음 | `prometheus.yml`이 컨테이너에 안 들어감 (`volumes` 확인) |
>
> **브라우저로 `http://localhost:8080/actuator/prometheus`를 직접 열어보는 것**이 가장 빠른 확인법입니다.
> 숫자가 잔뜩 나오면 앱 쪽은 정상입니다.

### 4) 트래픽 발생

```bash
# 10회 연속 호출
for i in {1..10}; do curl -s http://localhost:8080/hello; echo ""; sleep 0.2; done
```

> ⚠️ **500 에러가 나올 때까지 충분히 여러 번 호출하세요.**
> 지연이 800ms를 넘어야 에러가 나므로, **확률상 5번 중 1번쯤** 나옵니다.
> 에러 케이스가 있어야 **ERROR 로그와 Zipkin 에러 표시**를 확인할 수 있습니다.

### 5) Grafana에서 세 가지 확인

**`http://localhost:3000`** (admin / admin, 비밀번호 변경은 Skip 가능)

**Explore 메뉴(나침반 아이콘)** 에서 상단 드롭다운으로 데이터 소스를 바꿔가며 확인합니다.

#### ① Metrics — Prometheus 선택

- Select Metrics → **`api_hello_latency_seconds_count`** 선택 → Run query
- **시계열 그래프가 계단식으로 올라가면 성공** (Counter니까 누적입니다 → [02](02-메트릭.md))

#### ② Tracing — Zipkin 선택

- Traces 드롭다운 → `observability-service` 항목 중 하나 선택
- **Span 타임라인**이 나옵니다. 파란 막대를 클릭하면 상세 정보

> **Grafana의 Zipkin 화면은 개별 조회에 최적화되어 있습니다.**
> **전체 목록을 훑고 싶으면 Zipkin 전용 UI(`http://localhost:9411/zipkin/`)** 에서
> `RUN QUERY`를 누르는 편이 낫습니다. **느린 순으로 정렬해서 보기도 편합니다.**

#### ③ Logging — Loki 선택

1. 검색창 위의 **Label browser** 클릭
2. `app` → `observability-service` 선택 → **Show logs**
3. JSON 로그가 나오고, 안에 **`traceId`가 들어 있는 것**을 확인

**에러만 걸러 보기** — Builder를 **Code 모드**로 바꾸면 직접 질의를 쓸 수 있습니다.

```
{app="observability-service"} |= "ERROR"          # 본문에 "ERROR" 가 있는 줄
{app="observability-service", level="ERROR"}      # 라벨이 ERROR 인 줄 (더 빠름)
```

> **두 번째가 더 빠릅니다.** 라벨은 색인되어 있고, 본문 검색은 훑어야 하니까요. → [06 LogQL](06-분산-로깅.md)

### 6) 최종 확인 — 세 화면을 Trace ID로 잇기

**이게 이 실습의 진짜 목적입니다.**

```
① Zipkin 에서 느린(또는 에러난) Trace 하나를 고른다  →  Trace ID 복사
② Loki 에서 그 Trace ID 로 검색
        {app="observability-service"} |= "복사한_TraceID"
③ 그 요청이 남긴 로그만 정확히 뽑혀 나온다 ✅
```

**"지표에서 이상을 발견 → 추적으로 지점 특정 → 로그로 원인 확인"**
[01](01-관측-가능성이란.md)에서 말한 드릴다운을 직접 해보는 것입니다.

---

## 자주 만나는 문제

| 증상 | 원인 | 확인할 곳 |
|---|---|---|
| Prometheus Target이 `DOWN` | Actuator 미노출 / 포트 불일치 | `application.yml`의 `exposure.include` |
| 같음 (WSL 사용 중) | `host.docker.internal`이 WSL을 못 찾음 | `ip addr show eth0`의 IP로 교체 |
| Zipkin에 Trace가 안 보임 | **샘플링 10%** | `sampling.probability: 1.0` |
| 같음 | Zipkin 엔드포인트 주소 오타 | `management.zipkin.tracing.endpoint` |
| Loki에 로그가 안 옴 | Loki URL 오타 / Loki 미기동 | `docker compose ps` |
| 로그에 traceId가 비어 있음 | 추적 의존성 누락 | `micrometer-tracing-bridge-otel` |
| `@Timed` 메트릭이 안 생김 | `TimedAspect` 빈 미등록 또는 AOP 의존성 누락 | `ObserveConfig` |
| Grafana에 데이터소스가 없음 | 프로비저닝 경로 오류 | `volumes` 마운트 경로 |
| Grafana에서 "connection refused" | 데이터소스 URL을 `localhost`로 씀 | **서비스 이름**으로 바꾸기 |

> **주소 문제가 압도적으로 많습니다.** 헷갈릴 때는 이 표를 보세요.
>
> | 어디서 부르나 | 대상 | 주소 |
> |---|---|---|
> | 컨테이너 → 컨테이너 | Grafana → Prometheus | `http://prometheus:9090` |
> | 컨테이너 → 내 PC | Prometheus → Spring Boot | `host.docker.internal:8080` |
> | 내 PC → 컨테이너 | Spring Boot → Zipkin | `http://localhost:9411` |
> | 브라우저 → 컨테이너 | 나 → Grafana | `http://localhost:3000` |

## 다음 단계로 해볼 것

| # | 지금 | 해볼 것 |
|---|---|---|
| 1 | 서비스 1개 | **서비스 2개를 만들어 서로 호출** → 진짜 분산 추적 확인 |
| 2 | 대시보드 없음 | Spring Boot 공개 대시보드(ID `11378`, `19004`) **Import** |
| 3 | 알림 없음 | Grafana **알림 규칙** 추가 (에러율 5% 초과 시) |
| 4 | Zipkin | **Tempo**로 바꿔 Grafana 안에서 로그↔추적 자동 연결 |
| 5 | 직접 전송 | **OTel Collector**를 중간에 두기 → [05](05-opentelemetry.md) |

---

[← 분산 로깅](06-분산-로깅.md)
