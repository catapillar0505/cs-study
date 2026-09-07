> [← eda 목차](README.md)

# Apache Kafka

## 한 줄로

**로그**다. 소비해도 메시지가 사라지지 않고 **책갈피(오프셋)만 이동**한다. 그래서 재처리가 되고, 순차 I/O 덕에 압도적으로 빠르다.

## 정의

**분산 이벤트 스트리밍 플랫폼**

대용량 실시간 데이터 통합, 로그 수집, 이벤트 드리븐 백엔드를 위해 설계된 **분산 커밋 로그(Commit Log)** 형태의 플랫폼.

## 발상이 근본적으로 다르다

- **RabbitMQ = 큐**: 소비하면 사라진다
- **Kafka = 로그**: 사라지지 않는다

여기서 "로그"는 `System.out.println` 같은 로그가 아니라, **append-only 순차 기록 파일**을 뜻하는 컴퓨터과학 용어다. DB의 WAL(Write-Ahead Log), Git의 커밋 히스토리와 같은 개념이다. **뒤에만 붙이고, 중간을 수정하지 않고, 지우지 않는다.**

```
RabbitMQ (큐)
[Msg1][Msg2][Msg3]  ->  Consumer가 Msg1 소비  ->  [Msg2][Msg3]   (삭제됨)

Kafka (로그)
[Msg0][Msg1][Msg2][Msg3]   <- 그대로 남아 있음
   ▲          ▲
   │          └─ Consumer B의 offset (2까지 읽음)
   └─ Consumer A의 offset (0까지 읽음)
```

**Kafka는 메시지를 지우지 않고, Consumer가 "어디까지 읽었나"만 기록한다.**

## 5대 코어 컴포넌트

**Producer (발행처)**
이벤트를 생성해 특정 토픽으로 Push. **메시지 키(Message Key)** 를 지정해 파티션 배치를 통제할 수 있다.

**Topic (논리적 카테고리)**
이벤트를 분류하는 논리적 단위. DB의 테이블에 해당. 예: `order-events`, `user-events`.

**Partition (물리적 분할 큐)**
하나의 토픽을 물리적으로 여러 브로커에 쪼개 놓은 실제 저장 공간.
각 파티션은 **Append-only 로그 파일**로 동작한다.

```
Topic: order-events
├─ Partition 0  (Broker 1)  [0][1][2][3]
├─ Partition 1  (Broker 2)  [0][1][2]
└─ Partition 2  (Broker 3)  [0][1][2][3][4]
```

**왜 쪼개는가?** 토픽 하나가 파일 하나면 한 서버의 디스크 용량과 처리 속도가 한계가 된다. 파티션으로 쪼개 분산하면 용량과 처리량이 함께 늘어난다.

**Offset (순차 인덱스 식별자)**
파티션 내부에서 메시지에 부여되는 고유 순차 번호(0, 1, 2, ...).
Consumer가 어디까지 읽었는지 추적하는 절대적 위치 지표. **책갈피와 같다.** 책은 그대로 있고 사람마다 책갈피 위치가 다르다.

**Broker & Cluster**

- **Broker**: Kafka가 구동되는 개별 자바 프로세스(서버 인스턴스)
- **Cluster**: 여러 브로커가 네트워크로 묶여 복제(Replication)와 장애 허용(Fault Tolerance)을 달성하는 분산 구조

## 순서 보장의 범위 (매우 중요)

**오프셋은 파티션 안에서만 순차적이다. 따라서 순서 보장도 파티션 안에서만 된다.**
토픽 전체의 순서는 보장되지 않는다.

그래서 **메시지 키**가 있다.

```java
producer.send(new ProducerRecord<>("order-events", orderId, event));
//                                                  ↑ 키
```

**같은 키는 항상 같은 파티션으로 간다** (키의 해시 % 파티션 수).
`orderId`를 키로 주면 특정 주문의 이벤트들은 한 파티션에 순서대로 쌓이므로 순서가 보장된다.

## Consumer Group

```
Topic (파티션 3개)
├─ Partition 0 --> Consumer A ┐
├─ Partition 1 --> Consumer B ├─ Consumer Group "stock-service"
└─ Partition 2 --> Consumer C ┘
```

**핵심 규칙: 하나의 파티션은 동일 그룹 내에서 오직 하나의 Consumer만 소비한다.**

이 규칙이 순서 보장과 병렬 처리를 동시에 달성한다. 한 파티션을 둘이 읽으면 순서가 깨지므로 아예 막았다.

> **실무 함정: Consumer 개수는 파티션 개수를 넘을 수 없다.**
> 파티션이 3개인데 Consumer를 5개 띄우면 2개는 논다.
> **스케일 아웃의 상한선이 파티션 수**다. 파티션은 늘릴 수는 있지만 줄일 수는 없으므로 처음에 넉넉히 잡아야 한다.

**서로 다른 그룹은 독립적이다.**

```
Partition 0 --┬--> Group "stock-service"  (offset 100)
              └--> Group "analytics"      (offset 50)
```

같은 데이터를 재고 서비스와 분석 서비스가 각자의 속도로 읽는다. **그룹이 곧 Pub/Sub의 구독자 단위**다.

**Rebalance (리밸런스)**: Consumer가 추가되거나 죽으면 파티션 소유권이 재분배된다. 이 과정에서 잠시 소비가 멈춘다.

## 데이터 복제 (Replication)

각 파티션은 **Leader 1개 + Follower 여러 개**로 복제된다.

```
Partition 0
├─ Broker 1: Leader    <- 읽기/쓰기는 전부 여기로
├─ Broker 2: Follower  <- Leader를 복사만
└─ Broker 3: Follower
```

Leader 브로커가 다운되면 Follower 중 하나가 **Leader로 승격(Leader Election)** 되어 무중단 처리한다. 쿠버네티스의 ReplicaSet이 Pod를 복제해두는 것과 같은 발상이다. → [kubernetes/05](../../kubernetes/05-워크로드-deployment-replicaset-service.md)

## 왜 그렇게 빠른가

**순차 I/O (Sequential I/O)**
디스크는 랜덤 접근이 느릴 뿐 순차 접근은 느리지 않다. HDD 기준 랜덤 쓰기는 초당 수백 건이지만 **순차 쓰기는 초당 수백 MB**가 나온다. 헤드가 이동하지 않기 때문이다.
Kafka는 "뒤에만 붙인다"는 제약을 스스로 걸어 **모든 쓰기를 순차 쓰기로 만들었다.**

**페이지 캐시 활용**
JVM 힙에 캐시를 두지 않고 OS 페이지 캐시에 맡긴다. GC 부담이 없고, 방금 쓴 데이터는 메모리에 있어 읽기도 빠르다.

**Zero-copy**
디스크 → 소켓 전송 시 유저 공간을 거치지 않고 `sendfile()` 시스템 콜로 커널 내부에서 직접 보낸다. **유저/커널 모드 전환과 메모리 복사를 건너뛴다.**

## Pull 모델

- **RabbitMQ는 Push**: 브로커가 Consumer에게 밀어 넣는다
- **Kafka는 Pull**: Consumer가 브로커에 "새 메시지 있어?"라고 물어본다(polling)

Pull이 **백프레셔에 유리하다.** Consumer가 바쁘면 안 가져가면 되므로 자기 처리 능력을 스스로 조절한다. Push는 Consumer가 감당 못 할 속도로 밀어 넣으면 터진다. (RabbitMQ는 `prefetch` 설정으로 이를 제한한다.)

## 느슨한 소비 주기 (Commit & Retention)

메시지를 소비해도 즉시 지워지지 않고, Consumer는 자신이 어디까지 읽었는지 **Offset을 커밋(Commit)** 할 뿐이다.

따라서 비즈니스 로직에 오류가 발견되거나 로직을 고도화할 때 **오프셋을 과거 시점으로 리셋(Rewind)** 하여 데이터를 처음부터 몇 번이고 재처리할 수 있다.
나중에 만든 신규 서비스도 과거 데이터를 전부 읽을 수 있다.

보존 기간은 **Retention** 설정으로 정한다 (기본 7일, 무기한도 가능).

## 이름의 유래

작가 **프란츠 카프카(Franz Kafka)** 에서 따왔다. 개발자 Jay Kreps가 "쓰기에 최적화된 시스템이니 작가 이름을 붙이자"고 제안했고, 대학 때 카프카를 좋아했다고 한다. 기술적 의미는 없다.

---

[← RabbitMQ](04-rabbitmq.md) | [RabbitMQ vs Kafka →](06-rabbitmq-vs-kafka.md)
