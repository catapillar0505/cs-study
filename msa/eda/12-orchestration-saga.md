> [← eda 목차](README.md)

# Orchestration-based Saga

## 한 줄로

중앙 오케스트레이터가 각 서비스에 **명령(Command)** 을 하달하고 응답을 받아 다음 단계를 정한다. 흐름이 한곳에 모여 파악이 쉬운 대신 **SPOF**가 생긴다.

## 정의

중앙에 **오케스트레이터(Saga Orchestrator)** 를 두고, 각 서비스에 실행할 **명령(Command)** 을 하달하고 **응답(Reply)** 을 받아 다음 단계를 통제하는 방식.

서비스들이 자율적으로 이벤트를 주고받는 대신, 오케스트레이터라는 **중앙 상태 머신(State Machine)** 이 전체 트랜잭션의 상태와 순서를 독점 관리한다.

## 정상 흐름

```
               ┌───────────────────────┐
               │   Order Orchestrator  │  <- 전체 워크플로우 상태 관리
               └───┬───────────────┬───┘
    1. DeductStock │               │ 3. ProcessPayment
                   ▼               ▼
           ┌──────────────┐  ┌──────────────┐
           │    Stock     │  │   Payment    │
           │   Service    │  │   Service    │
           └──────────────┘  └──────────────┘
             2. 성공 응답        4. 성공 응답
```

1. **요청 수신**: Order Orchestrator가 주문 트랜잭션 시작
2. **명령 하달(Command)**: 메시지 큐를 통해 Stock Service에 `StockDeduct` 명령 전송
3. **응답 수신(Reply)**: Stock Service가 로컬 재고 차감 후 성공 응답 반환
4. **다음 단계**: 오케스트레이터가 Payment Service에 `ProcessPayment` 명령 전송

## 보상 흐름

```
                   ┌───────────────────────┐
                   │   Order Orchestrator  │  <- 2. Payment 실패 감지
                   └───────────┬───────────┘     3. 보상 명령 역발행 결정
            ┌──────────────────┴──────────────────┐
            │ 1. ProcessPayment (Fail)            │ 4. StockRestore
            ▼                                     ▼
    ┌──────────────┐                       ┌──────────────┐
    │   Payment    │                       │    Stock     │
    │   Service    │                       │   Service    │
    └──────────────┘                       └──────────────┘
```

1. Payment Service가 잔액 부족으로 실패 응답 반환
2. 오케스트레이터가 트랜잭션 상태를 `ROLLBACK_IN_PROGRESS`로 변경
3. Stock Service로 `StockRestore` 보상 명령 전송
4. 모든 보상 완료 후 주문 상태를 `CANCELLED`로 변경하고 종결

## Choreography와의 결정적 차이

- 서비스끼리 이벤트를 주고받지 **않는다.** 오케스트레이터와만 **1:1 통신**한다
- 주고받는 것이 이벤트가 아니라 **명령(Command)** 이다. "재고가 차감되었다"가 아니라 "재고를 차감해라"

> 이벤트와 명령의 구분은 [02 EDA의 기본 구조](02-eda의-기본-구조.md#이벤트-vs-명령-가장-중요한-개념-전환)

## 장단점

**장점**

| 항목 | 설명 |
|---|---|
| **명확한 상태 관리** | 오케스트레이터 DB만 조회하면 현재 진행 단계와 실패 원인을 한눈에 파악 |
| **순환 의존성 없음** | 별 모양(star) 구조라 서비스 간 순환 발생 불가 |
| 복잡도 통제 가능 | 조건 분기가 복잡해도 오케스트레이터 내부에서 통제 |

**단점**

| 항목 | 설명 |
|---|---|
| **SPOF 및 병목** | 오케스트레이터가 다운되면 전체 트랜잭션 마비. HA 클러스터링 필수 |
| 인프라 추가 필요 | Temporal, Camunda 등 워크플로우 엔진 또는 Spring StateMachine |
| 모놀리스 회귀 위험 | 자칫 오케스트레이터가 모든 비즈니스 로직을 흡수 |

**적합한 경우**: 참여 서비스 4개 이상, 예외 분기가 복잡, 진행 상태 실시간 모니터링이 필요한 시스템.

## 선택 기준 요약

| | Choreography | Orchestration |
|---|---|---|
| 서비스 수 | 2~3개 | **4개 이상** |
| 분기 복잡도 | 단순 | **복잡** |
| 흐름 파악 | 어려움 | **쉬움** |
| SPOF | 없음 | 있음 |
| 상태 조회 | 불가 | **가능** |

**작게 시작할 땐 Choreography, 복잡해지면 Orchestration**이 일반적인 진화 경로다.

---

[← Choreography-based Saga](11-choreography-saga.md) | [Transactional Outbox →](13-transactional-outbox.md)
