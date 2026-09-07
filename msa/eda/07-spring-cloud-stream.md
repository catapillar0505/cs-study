> [← eda 목차](README.md)

# Spring Cloud Stream

## 한 줄로

JDBC가 DB 벤더 차이를 숨기듯 **브로커 차이를 숨기는 계층**. 비즈니스 코드는 `Supplier`/`Function`/`Consumer` 자바 함수로만 쓰고, RabbitMQ↔Kafka는 **의존성 한 줄과 yml**로 바뀐다.

## 문제부터

Kafka로 개발하다가 RabbitMQ로 바꾸려면?

```java
// Kafka
@KafkaListener(topics = "order-events", groupId = "stock")
public void handle(ConsumerRecord<String, String> record) { ... }

// RabbitMQ — 코드를 전부 다시 씀
@RabbitListener(queues = "order-queue")
public void handle(Message message) { ... }
```

애플리케이션 코드가 **특정 브로커 기술에 결합**되었다.

## 해법: 한 겹 더 추상화

이미 익숙한 패턴이다. JDBC가 MySQL/Oracle 차이를 숨기고, JPA가 SQL 방언 차이를 숨기고, SLF4J가 Logback/Log4j 차이를 숨긴다.
**Spring Cloud Stream은 브로커 차이를 숨기는 계층이다.**

```
┌────────────────────────────────────────────┐
│  비즈니스 코드 (Supplier / Function / Consumer) │  <- 브로커를 모름
└──────────────┬─────────────────────────────┘
               │ Binding (논리 채널 이름)
┌──────────────▼─────────────────────────────┐
│              Binder (어댑터)                 │  <- 의존성으로 결정
└──────────────┬─────────────────────────────┘
┌──────────────▼─────────────────────────────┐
│         Message Broker (RabbitMQ / Kafka)   │
└────────────────────────────────────────────┘
```

**포터블 인프라**: 로컬 개발은 가벼운 RabbitMQ로, 운영은 Kafka로 전환해도 **애플리케이션 코드를 한 줄도 수정할 필요가 없다.** `application.yml`의 바인더 설정만 바꾸면 된다.

## 4대 핵심 개념

### (1) 메시지 브로커 추상화

브로커의 고유 스펙(Kafka의 Partition/Offset, RabbitMQ의 Exchange/Queue)을 프레임워크 수준에서 논리적으로 일치화한다.

### (2) Binder (인프라 어댑터)

실제 브로커와 Spring Boot 컨텍스트를 동적으로 매핑 연결하는 어댑터 엔진.
`build.gradle` 의존성에 따라 자동 탑재된다.

```gradle
implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
// 또는
implementation 'org.springframework.cloud:spring-cloud-starter-stream-kafka'
```

**의존성 한 줄만 바꾸면 브로커가 바뀐다.** JDBC 드라이버 교체와 동일한 구조.

### (3) Binding (선언적 연결 고리)

코드의 **논리적 채널 이름**과 브로커의 **물리적 큐/토픽 이름**을 연결한다.

```yaml
spring:
  cloud:
    stream:
      bindings:
        orderCreated-out-0:          # 논리 이름 (코드에서 사용)
          destination: order-events  # 물리 이름 (실제 토픽/Exchange)
        stockHandler-in-0:
          destination: order-events
          group: stock-service       # Consumer Group
```

**채널 이름 규칙**

```
{함수 빈 이름}-{in|out}-{인덱스}
```

- `in` = 입력(구독), `out` = 출력(발행)
- 인덱스는 0부터
- `Function<A,B> process()` 빈이 있으면 `process-in-0`, `process-out-0`이 자동 생성

### (4) 함수형 이벤트 프로그래밍 모델

Spring Cloud Stream 3.x/4.x부터 `@Input`, `@Output`, `@StreamListener`를 완전 배제(Deprecated)하고 **Java 8 표준 함수형 인터페이스**(`java.util.function`)를 사용한다.

| 역할 | 인터페이스 | 시그니처 | 의미 | 예시 |
|---|---|---|---|---|
| Producer | `Supplier<T>` | `() -> T` | 만들어서 내보냄 | "주문이 발생했다" 발행 |
| Consumer | `Consumer<T>` | `T -> void` | 받아서 처리, 끝 | "주문 메시지 받아 배송 시작" |
| Processor | `Function<T,R>` | `T -> R` | 받아서 가공, 다시 발행 | "주문 받아 → 포장 완료 발행" |

```java
@Bean
public Consumer<OrderCreatedEvent> stockHandler() {
    return event -> stockService.deduct(event.getOrderId());
}
```

**`@KafkaListener`도 `@RabbitListener`도 없다. 그냥 자바 함수다.**
브로커 관련 단어가 코드에 하나도 없다. 이것이 목표였다.

## StreamBridge — 즉석 발행

`Supplier`는 스케줄/트리거 기반 자동 발행용이다.
"주문 API 호출 시점에 발행"처럼 **명령형으로 발행**할 때는 `StreamBridge`를 쓴다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final StreamBridge streamBridge;

    public void createOrder(OrderRequest req) {
        Order order = orderRepository.save(Order.from(req));
        streamBridge.send("orderCreated-out-0", OrderCreatedEvent.from(order));
    }
}
```

> **주의**: 이 코드는 아직 안전하지 않다. `save()`는 DB에, `send()`는 브로커에 나가는 **이중 쓰기**다.
> 그 함정과 해법은 [13 Transactional Outbox](13-transactional-outbox.md).

---

[← RabbitMQ vs Kafka](06-rabbitmq-vs-kafka.md) | [메시지 신뢰성 →](08-메시지-신뢰성-ack-nack-dlq.md)
