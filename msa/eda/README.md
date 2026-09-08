# eda — 이벤트 기반 아키텍처

서비스를 쪼갠 뒤 **서비스끼리 어떻게 통신할 것인가**와 **트랜잭션이 서비스 경계를 넘지 못하는 문제**를 정리합니다.

[msa/05 서비스 간 통신](../05-서비스-간-통신.md)의 "비동기·메시지 브로커" 쪽과 [msa/08 분산 트랜잭션과 Saga](../08-분산-트랜잭션과-saga.md)를 실제 구현 수준까지 펼친 하위 폴더입니다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 00 | [용어 사전](00-용어-사전.md) | EDA·이벤트vs명령·브로커·AMQP 4대 구성요소·Exchange·컨슈머 그룹·파티션·오프셋·폴링·백프레셔·ACK/DLQ·전달 보장·멱등성·이중 쓰기 + 전체 색인 |
| 01 | [왜 비동기가 필요한가](01-왜-비동기가-필요한가.md) | 모놀리스의 두 축복, 동기 통신의 한계, 연쇄 장애 |
| 02 | [EDA의 기본 구조](02-eda의-기본-구조.md) | Producer·Broker·Consumer, 이벤트 vs 명령, 큐 vs 토픽 |
| 03 | [EDA 도입의 3대 이점](03-eda-도입의-3대-이점.md) | 공간적·시간적 결합 해제, 백프레셔, 선형 확장 |
| 04 | [RabbitMQ](04-rabbitmq.md) | AMQP, Exchange 3종, Binding·Routing Key, Management UI |
| 05 | [Apache Kafka](05-kafka.md) | 로그·파티션·오프셋, Consumer Group, 복제, 왜 빠른가 |
| 06 | [RabbitMQ vs Kafka](06-rabbitmq-vs-kafka.md) | 비교표와 선택 기준 |
| 07 | [Spring Cloud Stream](07-spring-cloud-stream.md) | Binder·Binding, 함수형 모델, StreamBridge |
| 08 | [메시지 신뢰성](08-메시지-신뢰성-ack-nack-dlq.md) | Ack/Nack, Poison Message, DLQ |
| 09 | [분산 트랜잭션 문제](09-분산-트랜잭션-문제.md) | `@Transactional`의 한계, 2PC를 안 쓰는 이유, 최종 일관성 |
| 10 | [Saga 패턴](10-saga-패턴.md) | 보상 트랜잭션, 롤백과의 차이, 보상 불가능한 작업 |
| 11 | [Choreography-based Saga](11-choreography-saga.md) | 이벤트 체인, 보상 전파, 추적 문제 |
| 12 | [Orchestration-based Saga](12-orchestration-saga.md) | 중앙 상태 머신, 명령·응답, SPOF |
| 13 | [Transactional Outbox](13-transactional-outbox.md) | 이중 쓰기 문제, outbox 테이블, Polling vs CDC |
| 14 | [멱등성](14-멱등성.md) | At-Least-Once, 이벤트 ID 기반 중복 체크 |
| 15 | [면접 대비 핵심 정리](15-면접-대비-핵심-정리.md) | 예상 질문과 답변 |
| 16 | [실습 로드맵](16-실습-로드맵.md) | 6단계 실습 순서, 로컬 리소스 주의사항 |

## 전체 구조 한눈에

```
모놀리스를 쪼갬
   │
   ├─ 문제 1: 통신이 강하게 결합되고 연쇄 장애가 전파됨
   │    └─ 해법: 메시지 브로커를 통한 비동기 통신 (EDA)
   │         ├─ RabbitMQ: 정교한 라우팅(Exchange), 소비 시 삭제, Push
   │         ├─ Kafka: 대용량, 로그로 보존, 재소비 가능, Pull
   │         ├─ Spring Cloud Stream: 브로커를 추상화 (Binder + Binding + 함수형)
   │         └─ Ack / Nack / DLQ: 메시지 유실 및 Poison Message 방지
   │
   └─ 문제 2: @Transactional이 서비스 경계를 넘지 못함
        └─ 해법: Saga 패턴 (보상 트랜잭션 + 최종 일관성)
             ├─ Choreography: 이벤트 체인. 단순하지만 추적 어려움
             ├─ Orchestration: 중앙 지휘. 복잡한 흐름에 적합, SPOF 존재
             └─ Outbox: DB 저장과 메시지 발행의 원자성 확보
                  └─ 대가: At-Least-Once -> 멱등성 필수
```

## 읽는 순서

배경지식이 없다면 [00 용어 사전](00-용어-사전.md)부터. 비유 중심이라 여기만 읽어도 나머지 문서가 읽힙니다.
이미 브로커를 써 봤다면 [01](01-왜-비동기가-필요한가.md)부터 순서대로 읽고, 용어가 막힐 때만 00으로 돌아오면 됩니다.

## 함께 보기

- 연쇄 장애를 **끊는** 쪽 (타임아웃·서킷브레이커) → [msa/06 장애 격리](../06-장애-격리.md)
- 이벤트가 서비스를 넘나들 때의 추적 → [msa/07 분산 추적](../07-분산-추적.md)
- 단일 DB 안에서의 트랜잭션·격리 수준 → [spring/08](../../spring/08-트랜잭션.md) · [database/mysql](../../database/mysql/README.md)
