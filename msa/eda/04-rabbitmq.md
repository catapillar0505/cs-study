> [← eda 목차](README.md)

# RabbitMQ

## 한 줄로

**큐**다. 소비하면 메시지가 사라진다. 대신 **Exchange**라는 분류소를 한 겹 더 둬서 라우팅이 매우 유연하다.

## 정의

**AMQP 0-9-1 표준 구현체**

**AMQP (Advanced Message Queuing Protocol)** 는 메시징의 표준 프로토콜이다. HTTP가 웹 통신의 표준인 것과 같다. AMQP를 따르는 브로커끼리는 클라이언트 라이브러리를 바꾸지 않고 교체 가능하다.

(Kafka는 AMQP를 쓰지 않는다. 자체 바이너리 프로토콜을 사용한다.)

## 왜 Exchange가 필요한가 (문제부터)

Producer가 큐에 직접 넣는다고 하면:

```java
queue.send("order-queue", msg);   // 큐 이름을 직접 지정
```

**Producer가 큐 이름을 알아야 한다.** 나중에 "재고 큐에도 보내고 통계 큐에도 보내야 해"가 되면 Producer 코드를 고쳐야 한다. 브로커를 뒀는데도 결합이 남는다.

그래서 RabbitMQ는 한 겹을 더 뒀다.

```
Producer --> Exchange --(Binding + Routing Key)--> Queue --> Consumer
             "우체국 분류소"                        "우편함"
```

**Producer는 Exchange에만 보낸다. 어느 큐로 갈지는 Exchange가 결정한다.**
라우팅 규칙이 코드가 아니라 **인프라 설정**으로 빠졌다. 새 Consumer 추가 시 큐를 만들고 바인딩만 하면 되고, Producer는 그대로다.

## 4대 코어 컴포넌트

```
┌───────────────┐
│   Producer    │  (이벤트 발행)
└───────┬───────┘
      Publish
        ▼
┌───────────────┐
│   Exchange    │  (라우팅 판정)
└───────┬───────┘
      Binding
   (Routing Key)
        ▼
┌───────────────┐
│     Queue     │  (메시지 적재)
└───────┬───────┘
      Consume
        ▼
┌───────────────┐
│    Consumer   │  (이벤트 처리)
└───────────────┘
```

| 컴포넌트 | 역할 | 비유 |
|---|---|---|
| **Producer** | 브로커로 메시지 Push. **큐에 직접 넣지 않고 반드시 Exchange를 거침** | 편지 부치는 사람 |
| **Exchange** | 바인딩 규칙과 라우팅 키를 대조해 타겟 큐로 릴레이 | 우체국 분류소 |
| **Queue** | Consumer가 소비할 때까지 메시지 보관 (메모리 또는 디스크) | 우편함 |
| **Consumer** | 큐를 구독하다 메시지 도착 시 비즈니스 로직 실행 | 편지 꺼내 읽는 사람 |

추가 개념 2가지:

- **Binding**: Exchange와 Queue를 잇는 **규칙**
- **Routing Key**: 메시지에 붙는 **주소 라벨** (Producer가 지정)

## Exchange 3대 라우팅 전략

### Direct Exchange — 키 정확 일치 (Exact Match)

```
Routing Key: "order.create"
Binding Key: "order.create" -> order-queue      매칭 O
Binding Key: "order.cancel" -> cancel-queue     매칭 X
```

> **흔한 오해**: Direct를 "1:1"로 설명하는 자료가 많지만 정확하지 않다.
> **같은 라우팅 키로 여러 큐를 바인딩하면 Direct도 여러 큐로 보낸다.**
> Direct의 본질은 "1:1"이 아니라 **"키 정확 일치"** 다.

### Fanout Exchange — 브로드캐스트

라우팅 키를 **완전히 무시하고** 바인딩된 모든 큐에 복사 전파한다.

```
[ Fanout Exchange ]
  No Routing Key
         │
         ├--> Queue A (order)
         ├--> Queue B (payment)
         └--> Queue C (shipping)
```

`fan out`은 선풍기 날개가 퍼지듯 **하나가 여러 갈래로 퍼진다**는 뜻. 전자공학에서 신호 하나가 여러 소자로 갈라지는 것을 부르던 용어다.

용도: 로깅, 통계, 캐시 무효화 등 다중 시스템 동시 전파.

### Topic Exchange — 패턴 매칭

라우팅 키를 `order.create.success`처럼 **점으로 구분된 단어들**로 쓰고 와일드카드로 매칭한다.

| 와일드카드 | 의미 | 예시 |
|---|---|---|
| `*` (asterisk) | **정확히 한 단어** | `order.*` → `order.create` O, `order.create.fail` X |
| `#` (hash) | **0개 이상의 단어** | `order.#` → `order`, `order.create`, `order.create.fail` 모두 O |

`#.error`로 바인딩하면 모든 서비스의 에러 이벤트만 골라 받을 수 있다.
**Direct와 Fanout의 중간이면서 가장 유연해 실무에서 가장 많이 쓴다.**

## Management UI

RabbitMQ는 내장 관리 포털을 제공한다.

| 포트 | 용도 |
|---|---|
| **5672** | AMQP 프로토콜 (애플리케이션 접속) |
| **15672** | Management UI (웹 브라우저) |

실시간 큐 메시지 누적량(Message Backlog), 초당 처리율(Throughput), 커넥션 상태를 시각적으로 모니터링할 수 있다.

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management
```

기본 계정은 `guest` / `guest` (localhost에서만 접속 가능).

---

[← EDA 도입의 3대 이점](03-eda-도입의-3대-이점.md) | [Apache Kafka →](05-kafka.md)
