> [← eda 목차](README.md)

# 메시지 신뢰성 (Ack / Nack / DLQ)

## 한 줄로

**Ack를 받기 전까지 브로커는 메시지를 지우지 않는다.** 다만 "실패하면 무조건 재시도"는 재앙이므로, 몇 번 실패한 메시지는 **DLQ로 격리**해 정상 트래픽을 흐르게 한다.

## 문제

Consumer가 메시지를 받아서 처리하다가 죽으면? **메시지가 증발한다.**
주문은 들어왔는데 재고 차감이 안 된 상태로 영원히 남는다.

## Ack (Acknowledge, 수신 완료 승인)

**"처리를 완료했다"고 명시적으로 알려주기 전까지 브로커는 메시지를 지우지 않는다.**

```
1. 브로커 -> Consumer로 메시지 전달 (큐에 보관 유지, "미확인" 표시)
2. Consumer가 처리 수행
3. Consumer -> 브로커로 Ack 전송
4. 브로커가 메시지를 큐에서 완전히 제거(De-queue)
```

**2번 도중 Consumer가 죽으면?** Ack가 오지 않으므로 브로커가 **다른 Consumer에게 재전달**한다. 메시지가 유실되지 않는다.

택배 수령 사인과 같다. 사인을 받아야 배송 완료 처리를 한다.

> 재전달이 곧 **중복 수신**이다. 그래서 [14 멱등성](14-멱등성.md)이 따라온다.

## Nack (Negative Acknowledge, 수신 거부)

처리에 실패했을 때 보내는 신호. 두 가지 선택지가 있다.

- **Re-queue**: 큐로 되돌려 다시 시도
- **DLQ로 이동**: 포기하고 격리

## 핵심 함정: 무한 재시도 루프와 Poison Message

**"실패하면 무조건 큐로 되돌린다"** 는 재앙이다.

메시지 자체가 잘못돼 있으면(JSON 파싱 불가, 필수 필드 누락) **몇 번을 재시도해도 항상 실패한다.**
큐로 돌아오면 다시 시도하고, 또 실패하고, 또 돌아오고... **무한 루프**에 빠진다.

이런 메시지를 **독이 든 메시지(Poison Message)** 라고 한다.
큐 맨 앞에 박혀 있으면 **뒤의 정상 메시지들이 전부 막혀 큐 하나가 통째로 마비된다.**

## DLQ (Dead Letter Queue)

**"몇 번 시도해도 안 되면 별도 큐로 격리한다."**

```
      [ Producer ]
            │ (1. Publish)
            ▼
    ┌───────────────┐
    │   Exchange    │<─────────────────────────┐
    └───────┬───────┘                          │
            │ (2. Routing)                     │ (5. Re-Publish)
            ▼                                  │
    ┌───────────────┐            ┌─────────────┴─────────┐
    │  Main Queue   │            │   Dead Letter Queue   │
    └───────▲───────┘            └───────────▲───────────┘
            │ (3. Subscribe)                 │
            │                                │ (4-B. Nack / 재시도 초과)
    ┌───────┴───────┐                        │
    │   Consumer    ├────────────────────────┘
    └───────┬───────┘
            │ (4-A. Ack)
            ▼
      [ 처리 성공 ]
```

**Dead Letter(사서함 불명 우편)** 는 실제 우편 용어다.
주소 불명이나 수취인 불명으로 배달하지 못한 편지를 모아두는 **반송 우편물 취급소(Dead Letter Office)** 에서 왔다. 미국 우체국에 실제로 있는 부서다.

**핵심 효과**: 문제 메시지를 빼내어 **정상 트래픽이 계속 흐르게** 한다.
격리된 메시지는 나중에 원인을 수정하고 **재발행(Re-publish)** 하여 정상 파이프라인으로 회수한다.

## Spring Cloud Stream 설정

```yaml
spring:
  cloud:
    stream:
      bindings:
        stockHandler-in-0:
          destination: order-events
          group: stock-service
          consumer:
            max-attempts: 3                    # 총 3회 시도
            back-off-initial-interval: 1000    # 1초 후 재시도
            back-off-multiplier: 2.0           # 지수 백오프
      rabbit:
        bindings:
          stockHandler-in-0:
            consumer:
              auto-bind-dlq: true       # DLQ 자동 생성
              republish-to-dlq: true    # 실패 원인 헤더 포함해 DLQ 발행
```

> **DLQ 모니터링은 필수다.** DLQ에 메시지가 쌓이는데 아무도 보지 않으면 데이터가 조용히 유실되는 것과 같다. 반드시 알림을 설정해야 한다.

---

[← Spring Cloud Stream](07-spring-cloud-stream.md) | [분산 트랜잭션 문제 →](09-분산-트랜잭션-문제.md)
