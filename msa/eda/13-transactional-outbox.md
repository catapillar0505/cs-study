> [← eda 목차](README.md)

# Transactional Outbox 패턴

## 한 줄로

**DB 저장과 메시지 발행은 서로 다른 시스템**이라 한 트랜잭션으로 묶이지 않는다. 브로커로 직접 보내지 말고 **같은 DB의 outbox 테이블에 함께 저장**한 뒤, 별도 Relay가 읽어 발행한다.

## 문제: 이중 쓰기 (Dual Write)

지금까지 배운 것을 합치면 이런 코드가 나온다.

```java
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(order);                        // ① DB
    streamBridge.send("orderCreated-out-0", event);     // ② 브로커
}
```

**안전해 보이지만 치명적인 버그다.**

이유: **①과 ②는 서로 다른 시스템**이다. `@Transactional`은 DB에만 걸려 있고 **브로커는 트랜잭션 밖**이다.

**시나리오 A — DB 커밋 성공, 발행 실패**

```
① save() 성공 -> 커밋
② send() 실패 (브로커 다운, 네트워크 순단)
결과: 주문은 있는데 재고 차감이 영원히 안 됨
```

**시나리오 B — 발행 성공, DB 커밋 실패**

```
② send() 성공 -> Stock이 재고를 차감
① 커밋 직전 예외 -> 롤백
결과: 주문은 없는데 재고만 깎임 (데이터 오염)
```

> 참고: B 시나리오는 `@TransactionalEventListener(phase = AFTER_COMMIT)`으로 일부 막을 수 있다.
> 하지만 **A 시나리오는 못 막는다.** 커밋 직후 서버가 죽으면 발행이 영영 안 된다.

## 핵심 개념

> **브로커로 직접 보내지 말고, 같은 DB의 OUTBOX 테이블에 함께 저장한다.**

```java
@Transactional
public void createOrder(OrderRequest req) {
    orderRepository.save(order);         // ① orders 테이블
    outboxRepository.save(outboxEvent);  // ② outbox 테이블 (같은 DB!)
}   // 하나의 트랜잭션 = 둘 다 되거나 둘 다 안 되거나
```

**같은 DB이므로 하나의 트랜잭션으로 묶인다. ACID가 보장된다.**
"비즈니스 데이터 + 발행할 메시지 데이터"를 한 트랜잭션으로 커밋하는 것이 전부다.

## 이름의 유래

**Outbox(발신함)** 는 사무실 책상의 그것이다.
편지를 직접 우체국에 가져가지 않고 **발신함에 넣어두면 나중에 우편 담당자가 수거해 부친다.**
내 일은 발신함에 넣는 것까지이고, 발송은 다른 사람의 책임이다.

## 아키텍처 구조

```
[ Application ]
       │  1. 하나의 DB 트랜잭션 내에서 처리 (@Transactional)
       ▼
┌──────────────────────────────────────────┐
│                 MySQL DB                 │
│  ┌─────────────────┐ ┌────────────────┐  │
│  │  orders (Table) │ │ OUTBOX (Table) │  │
│  └─────────────────┘ └────────────────┘  │
└─────────────────────┬────────────────────┘
                      │  2. 미발행 이벤트 읽기 (Polling 또는 CDC)
                      ▼
              [ Message Relay ]
                      │  3. 메시지 발행
                      ▼
             [ RabbitMQ / Kafka ]
```

**처리 단계**

1. **트랜잭션 저장**: `orders`에 주문 INSERT + `outbox`에 이벤트 JSON을 `status = 'PENDING'`으로 INSERT
2. **메시지 추출**: 별도 프로세스(Message Relay)가 `PENDING` 상태 메시지를 읽음
3. **발행 및 완료 처리**: 브로커로 전송하고 ACK를 받으면 `PROCESSED`로 수정하거나 행 삭제

## Message Relay 구현 2가지

| 구분 | Polling Publisher | CDC (Change Data Capture) |
|---|---|---|
| **원리** | 스케줄러가 주기적으로 DB 조회<br>`SELECT * FROM outbox WHERE status='PENDING'` | DB 트랜잭션 로그(binlog, WAL)를 실시간 감지 |
| **대표 기술** | Spring Scheduler, Quartz | Debezium, Kafka Connect |
| **장점** | 구현이 간단, 추가 인프라 불필요 | **DB 부하 없음, 밀리초 단위 실시간성** |
| **단점** | 주기적 조회 부하, 폴링 주기만큼 지연 | 별도 CDC 인프라 구축 필요 |

**Polling 구현 예시**

```java
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishPendingEvents() {
    List<OutboxEvent> events = outboxRepo.findTop100ByStatusOrderByCreatedAt(PENDING);
    for (OutboxEvent e : events) {
        streamBridge.send(e.getChannel(), e.getPayload());
        e.markProcessed();
    }
}
```

**CDC 보충**: MySQL의 binlog는 DB가 복제(replication)용으로 이미 남기고 있는 변경 이력이다. Debezium이 이를 실시간으로 읽어 Kafka로 보낸다. 애플리케이션이 DB에 추가 쿼리를 날리지 않으므로 부하가 없다.

**학습 단계에서는 Polling으로 충분하다.**

> 그런데 Outbox를 써도 완벽하지 않다. Relay가 발행 후 상태 업데이트 전에 죽으면 **중복 발행**된다. → [14 멱등성](14-멱등성.md)

---

[← Orchestration-based Saga](12-orchestration-saga.md) | [멱등성 →](14-멱등성.md)
