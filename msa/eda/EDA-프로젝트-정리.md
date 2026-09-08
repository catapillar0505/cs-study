# Spring Cloud Stream 기반 EDA 실습 완전 정리
### `eda-project` (이벤트 발행/구독 기초) + `eda-project2` (SAGA 분산 트랜잭션)

> **이 문서의 대상**: 스프링 부트로 웹 애플리케이션은 만들어봤지만, 메시지 큐 / RabbitMQ / MSA / 이벤트 기반 아키텍처(EDA) / 분산 트랜잭션은 처음인 사람.
>
> **읽는 법**: Part 0~2는 "왜 이런 게 필요한가"에 대한 배경 지식입니다. 코드가 급하면 Part 4부터 읽되, 모르는 용어가 나오면 Part 8 용어 사전과 앞부분으로 되돌아오세요.

---

## 목차

- [Part 0. 왜 이벤트인가 — 동기 호출의 한계](#part-0-왜-이벤트인가--동기-호출의-한계)
- [Part 1. 기반 기술 기초 — 메시지 브로커와 RabbitMQ](#part-1-기반-기술-기초--메시지-브로커와-rabbitmq)
- [Part 2. Spring Cloud Stream 이해하기](#part-2-spring-cloud-stream-이해하기)
- [Part 3. 두 프로젝트의 공통 뼈대](#part-3-두-프로젝트의-공통-뼈대)
- [Part 4. eda-project #1 — 이벤트 발행과 구독의 기본기](#part-4-eda-project-1--이벤트-발행과-구독의-기본기)
- [Part 5. eda-project2 — SAGA 패턴으로 분산 트랜잭션 다루기](#part-5-eda-project2--saga-패턴으로-분산-트랜잭션-다루기)
- [Part 6. 두 프로젝트 비교](#part-6-두-프로젝트-비교)
- [Part 7. 이 구조의 한계와 다음 단계](#part-7-이-구조의-한계와-다음-단계)
- [Part 8. 용어 사전](#part-8-용어-사전)
- [Part 9. 실행·검증 치트시트](#part-9-실행검증-치트시트)

---

# Part 0. 왜 이벤트인가 — 동기 호출의 한계

## 0-1. 출발점: 주문과 재고는 서로 다른 서비스다

우리가 만들 시스템은 아주 단순합니다.

> 손님이 "사과 2개 주문"을 하면 → 주문을 저장하고 → 재고를 2개 깎는다.

모놀리식(하나의 애플리케이션)이라면 이건 고민할 게 없습니다. 하나의 메서드 안에서 `orderRepository.save()` 하고 `stockRepository.decrease()` 하면 끝이고, 둘 중 하나라도 실패하면 `@Transactional`이 통째로 롤백해 줍니다. **DB 하나 = 트랜잭션 하나**이기 때문입니다.

그런데 MSA(마이크로서비스 아키텍처)에서는 주문 서비스와 재고 서비스가 **다른 프로세스, 다른 서버, 다른 DB**입니다. 이 순간 위의 편안함이 전부 사라집니다.

## 0-2. 첫 번째 시도: REST로 직접 호출하면 안 되나?

```java
// order-service
@Transactional
public void createOrder(...) {
    orderRepository.save(order);
    restClient.post("http://stock-service/api/stocks/decrease", ...); // 동기 호출
}
```

동작은 합니다. 하지만 네 가지 심각한 문제가 있습니다.

| # | 문제 | 설명 |
|---|---|---|
| 1 | **강한 결합 (Coupling)** | 주문 서비스가 재고 서비스의 **주소와 API 스펙을 알아야** 합니다. 나중에 "주문되면 포인트도 적립, 알림도 발송, 통계도 집계"가 추가되면 주문 서비스 코드를 그때마다 고쳐야 합니다. |
| 2 | **가용성이 곱셈이 된다** | 주문 서비스 가용성 99%, 재고 서비스 99% → 전체 가용성은 99% × 99% = 98%. 재고 서비스가 죽으면 **주문 자체를 못 받습니다.** 서비스를 쪼갤수록 더 잘 죽는 시스템이 됩니다. |
| 3 | **지연이 누적된다** | 주문(50ms) → 재고(80ms) → 알림(120ms)을 순서대로 기다리면 사용자는 250ms를 기다립니다. 뒤에 붙는 기능이 늘수록 응답이 계속 느려집니다. |
| 4 | **트랜잭션이 깨진다** | `restClient.post()`가 성공했는데 그 다음 줄에서 예외가 나면? 내 DB는 롤백되지만 **이미 깎인 남의 재고는 되돌아오지 않습니다.** |

## 0-3. 발상의 전환: "시켜라"가 아니라 "알려라"

동기 호출은 **명령(Command)** 입니다.

> 주문 서비스 → 재고 서비스: **"재고 2개 깎아. 그리고 결과 알려줄 때까지 기다릴게."**

이벤트 기반(EDA)은 **사실의 통보(Event)** 입니다.

> 주문 서비스 → 세상 전체: **"주문이 생성되었음. (누가 뭘 하든 나는 관심 없음)"**

이 한 줄의 차이가 모든 것을 바꿉니다.

| | 명령 (Command) | 이벤트 (Event) |
|---|---|---|
| 이름 형태 | 명령형: `DecreaseStock` | **과거형**: `OrderCreated`, `StockReserved` |
| 받는 사람 | **정확히 1명**을 지정 | **몇 명이든** (0명도 가능) |
| 보낸 쪽의 기대 | "이걸 해줘" (결과에 관심 있음) | "이런 일이 있었어" (누가 듣든 상관 없음) |
| 거절 가능성 | 받는 쪽이 거절/실패할 수 있음 | **이미 일어난 사실**이라 거절 불가 |
| 결합도 | 높음 | 낮음 |

> 💡 **핵심 원리**: 이벤트 이름을 반드시 **과거형**으로 짓는 이유가 여기 있습니다. `OrderCreatedEvent`는 "주문을 생성해라"가 아니라 **"주문이 이미 생성되었다는 변경 불가능한 사실"** 입니다. 이 사실을 듣고 무엇을 할지는 **듣는 쪽(Consumer)이 스스로 결정**합니다. 이것을 **수신자 주도(Receiver-driven)** 설계라고 합니다.

## 0-4. 그래서 EDA가 얻는 것

1. **결합 제거** — 주문 서비스는 재고 서비스의 존재조차 모릅니다. 나중에 포인트 서비스가 추가돼도 **주문 서비스 코드는 한 줄도 안 바뀝니다.** (구독자만 추가하면 됨)
2. **가용성 향상** — 재고 서비스가 죽어 있어도 주문은 정상 접수됩니다. 메시지는 큐에 쌓여 있다가, 재고 서비스가 살아나면 그때 처리됩니다. 이걸 **시간적 분리(Temporal Decoupling)** 라고 합니다.
3. **응답 속도** — 주문 서비스는 메시지를 브로커에 던지는 즉시(수 ms) 사용자에게 응답합니다.
4. **부하 완충 (Buffering)** — 초당 1만 건이 몰려도 큐가 완충 역할을 합니다. 재고 서비스는 자기 속도대로 처리하면 됩니다.

## 0-5. 대신 잃는 것 (공짜는 없다)

| 잃는 것 | 의미 |
|---|---|
| **강한 일관성** | "주문 완료" 응답을 받은 직후 재고를 조회하면 아직 안 깎여 있을 수 있습니다. 잠시 후엔 맞아집니다 → **최종적 일관성(Eventual Consistency)** |
| **간단한 디버깅** | 흐름이 코드에 한 줄로 안 보입니다. 로그가 여러 서비스에 흩어집니다. |
| **간단한 롤백** | `@Transactional` 하나로 전체를 되돌릴 수 없습니다 → **Part 5의 SAGA 패턴이 필요한 이유** |
| **인프라 부담** | 브로커(RabbitMQ)라는 새 구성요소를 운영해야 합니다. |

**두 실습 프로젝트는 정확히 이 순서로 배웁니다.**

- `eda-project` : 0-4의 "얻는 것"을 구현 → **이벤트 발행/구독의 기본기**
- `eda-project2` : 0-5의 "잃는 것" 중 **롤백 문제를 SAGA로 해결**

---

# Part 1. 기반 기술 기초 — 메시지 브로커와 RabbitMQ

## 1-1. 메시지 브로커란

**메시지 브로커(Message Broker)** 는 서비스들 사이에 놓인 **우체국**입니다.

```
[보내는 쪽]  ──메시지──▶  [ 브로커 ]  ──메시지──▶  [받는 쪽]
 Producer                  RabbitMQ                Consumer
 (발행자)                  (우체국)                 (구독자)
```

- 보내는 쪽은 **받는 쪽이 살아있는지 몰라도** 됩니다. 브로커에 넣으면 끝.
- 받는 쪽은 **자기가 준비됐을 때** 꺼내갑니다.
- 브로커는 메시지를 **디스크에 저장**할 수 있어, 브로커가 재시작돼도 메시지가 살아남습니다.

이 프로젝트는 브로커로 **RabbitMQ**를 씁니다. (`docker-compose.yml`로 띄웁니다)

> 참고: 대안으로 **Kafka**가 있습니다. RabbitMQ는 "우체국"(꺼내가면 메시지가 사라짐), Kafka는 "로그 파일"(읽어도 남아있고, 여러 번 되감아 읽을 수 있음)에 가깝습니다. Spring Cloud Stream을 쓰면 **코드를 안 바꾸고 의존성만 교체해서** 둘 사이를 옮겨 다닐 수 있습니다 (Part 2 참고).

## 1-2. AMQP와 RabbitMQ의 4대 구성요소 ★가장 중요★

RabbitMQ는 **AMQP(Advanced Message Queuing Protocol)** 라는 표준 프로토콜을 구현한 브로커입니다. AMQP의 핵심은 **"발행자는 큐에 직접 넣지 않는다"** 입니다.

```
                     ┌──────────────────────────────────────────┐
                     │              RabbitMQ                     │
                     │                                           │
  Producer ─────────▶│  ┌──────────┐  binding   ┌──────────┐    │──────▶ Consumer A
 (routing key:       │  │ Exchange │───(key)───▶│  Queue A │    │
  "order.created")   │  │ (교환기)  │            └──────────┘    │
                     │  │          │            ┌──────────┐    │──────▶ Consumer B
                     │  │          │───(key)───▶│  Queue B │    │
                     │  └──────────┘            └──────────┘    │
                     └──────────────────────────────────────────┘
```

| 구성요소 | 우체국 비유 | 역할 |
|---|---|---|
| **Producer** | 편지 보내는 사람 | 메시지를 만들어 Exchange로 보냅니다. **큐 이름을 모릅니다.** |
| **Exchange (익스체인지)** | 우체국 **분류기** | 메시지를 받아서 **규칙에 따라** 어느 큐로 넣을지 결정합니다. 저장하지 않습니다. 매칭되는 큐가 하나도 없으면 **메시지는 버려집니다.** |
| **Binding (바인딩)** | 분류 **규칙표** | "Exchange X로 온 메시지 중 `order.*` 패턴은 Queue A로" 같은 연결 규칙. **Routing Key**를 조건으로 씁니다. |
| **Queue (큐)** | **우편함** | 메시지가 실제로 **쌓여서 대기**하는 곳. 소비자가 꺼내갈 때까지 보관합니다. |
| **Consumer** | 우편함 주인 | 큐에서 메시지를 꺼내 처리합니다. |

> 💡 **왜 Exchange를 거치게 만들었나?**
> Producer가 큐에 직접 넣으면, 나중에 "이 메시지를 다른 서비스도 받아야 해"가 되었을 때 **Producer 코드를 고쳐야** 합니다. Exchange를 두면, 새 구독자는 **자기가 알아서 새 큐를 만들고 Exchange에 바인딩**하면 됩니다. Producer는 아무것도 모릅니다. → **0-4에서 말한 "결합 제거"가 실제로 구현되는 지점**이 바로 여기입니다.

## 1-3. Routing Key와 Exchange 타입

**Routing Key(라우팅 키)** 는 메시지에 붙는 **주소 라벨**입니다. 예: `order.created`, `order.cancelled`, `payment.completed`

Exchange는 타입에 따라 이 라벨을 다르게 해석합니다.

| Exchange 타입 | 규칙 | 예시 |
|---|---|---|
| **Direct** | 라우팅 키가 **정확히 일치**해야 함 | 바인딩 키 `order.created` = 라우팅 키 `order.created` |
| **Topic** ★ | **와일드카드 패턴** 매칭 | 바인딩 키 `order.*`가 `order.created`, `order.cancelled`를 모두 잡음 |
| **Fanout** | 라우팅 키 **무시**, 바인딩된 **모든 큐에 복사** | 방송(broadcast) |
| **Headers** | 라우팅 키 대신 헤더 값으로 매칭 | 거의 안 씀 |

**Spring Cloud Stream의 RabbitMQ 바인더는 기본적으로 Topic Exchange를 만듭니다.** 이 프로젝트도 전부 Topic입니다.

Topic 와일드카드 규칙 (구분자는 점 `.`):

| 기호 | 의미 | `order.created`에 매칭? | `order.item.created`에 매칭? |
|---|---|---|---|
| `order.*` | `*` = **정확히 한 단어** | ✅ 매칭 | ❌ (단어가 3개) |
| `order.#` | `#` = **0개 이상의 단어** | ✅ 매칭 | ✅ 매칭 |
| `#` | 전부 | ✅ | ✅ |

👉 그래서 `eda-project`의 재고 서비스 설정 `binding-routing-key: 'order.*'` 는 **"order로 시작하는 한 단계 하위 이벤트는 다 받겠다"** 는 뜻입니다. 나중에 `order.cancelled` 이벤트가 추가돼도 **설정을 안 고쳐도 자동으로 수신**됩니다.

## 1-4. 컨슈머 그룹 = 경쟁 소비자 패턴 ★중요★

재고 서비스를 트래픽 때문에 **3대로 늘렸다**고 합시다. 주문 이벤트 1건이 오면 어떻게 돼야 할까요?

- ❌ 3대가 모두 받으면 → **재고가 3번 깎임 (재앙)**
- ✅ 3대 중 **딱 1대만** 받아야 함

이걸 **경쟁 소비자 패턴(Competing Consumers)** 이라 하고, Spring Cloud Stream에서는 `group` 설정 한 줄로 구현합니다.

```yaml
stockControl-in-0:
  destination: order-created-topic   # Exchange 이름
  group: stock-group                 # ★ 컨슈머 그룹
```

**동작 원리**: `group`을 주면 RabbitMQ에 **`order-created-topic.stock-group`** 이라는 **이름 있는 물리적 큐 1개**가 만들어집니다. 재고 서비스 3대는 **같은 큐 하나**를 나눠 먹습니다. 큐에서 메시지를 꺼내가는 건 원래 한 번에 한 소비자뿐이므로, 자연스럽게 중복이 방지됩니다.

```
                         ┌──────────────────────────────────┐
                         │  order-created-topic (Exchange)  │
                         └───────┬──────────────────┬───────┘
                                 │                  │
              ┌──────────────────▼──────┐   ┌───────▼─────────────────┐
              │ .stock-group (큐)        │   │ .point-group (큐)        │
              └───┬──────┬──────┬────────┘   └──────────┬──────────────┘
                  │      │      │  (셋 중 하나만)         │
              재고1    재고2   재고3                   포인트 서비스
              ── 같은 그룹 = 나눠 먹기 ──          ── 다른 그룹 = 각자 다 받음 ──
```

| | `group` **있음** | `group` **없음** |
|---|---|---|
| 만들어지는 큐 | `목적지.그룹명` — **이름 있고 영구적** | 인스턴스마다 **익명 임시 큐** (auto-delete) |
| 인스턴스 3대일 때 | 1건 → **1대만** 처리 | 1건 → **3대 전부** 처리 (브로드캐스트) |
| 서비스가 꺼져 있는 동안 온 메시지 | 큐에 **쌓여 있다가** 재시작 후 처리 | 큐가 사라져서 **유실** |
| 쓰는 상황 | 대부분의 업무 처리 ★ | 각 인스턴스의 로컬 캐시 갱신 등 |

> ⚠️ **실무에서 가장 흔한 사고**: `group`을 안 주고 배포했다가 서비스 재시작 사이에 온 메시지를 통째로 잃어버리는 것. **업무 처리 컨슈머에는 반드시 `group`을 주세요.**
>
> 그리고 **"그룹이 다르면 각자 다 받는다"** 는 성질이 바로 EDA의 확장 포인트입니다. 포인트 서비스는 `point-group`으로 같은 Exchange를 구독하면, 주문 서비스는 아무것도 모른 채 기능이 하나 늘어납니다.

## 1-5. ACK, 재시도, 그리고 DLQ

컨슈머가 메시지를 꺼내갔는데 **처리 중 예외가 나면** 어떻게 될까요?

**ACK(Acknowledgement, 확인 응답)** 개념이 여기서 나옵니다.

1. 컨슈머가 큐에서 메시지를 꺼냅니다. 이때 메시지는 아직 큐에서 **완전히 지워지지 않고 "처리 중" 상태**가 됩니다.
2. 처리에 성공하면 컨슈머가 브로커에 **ACK**를 보냅니다 → 그제서야 큐에서 삭제됩니다.
3. 처리에 실패(예외)하면 **NACK** → 브로커는 메시지를 **다시 큐에 넣거나(requeue)**, 다른 곳으로 보냅니다.

여기서 위험한 상황이 생깁니다. 코드에 버그가 있어서 **영원히 실패하는 메시지**가 있다면? 무한히 재시도되며 **CPU를 태우고 뒤의 메시지를 막습니다.** 이걸 **독약 메시지(Poison Message)** 라고 합니다.

이 프로젝트는 두 겹의 방어를 씁니다.

```yaml
consumer:
  max-attempts: 3          # ① 애플리케이션 메모리 안에서 3번까지 시도
rabbit:
  bindings:
    stockControl-in-0:
      consumer:
        auto-bind-dlq: true      # ② 3번 다 실패하면 격리 큐(DLQ)로 보냄
        republish-to-dlq: true   # ③ 보낼 때 에러 스택트레이스를 헤더에 첨부
```

**실제 실행 순서**:

```
메시지 도착
   │
   ├─ 1회차 시도 → 예외 💥
   ├─ 2회차 시도 → 예외 💥      ← ① max-attempts: 3 (첫 시도 포함 총 3회)
   ├─ 3회차 시도 → 예외 💥         이 재시도는 브로커를 안 거치고
   │                                애플리케이션 메모리 안에서 반복됩니다
   ▼
DLQ로 이동: order-created-topic.stock-group.dlq   ← ②
   (메시지 헤더에 x-exception-stacktrace 등이 붙음) ← ③
   │
   ▼
운영자가 나중에 DLQ를 열어보고 원인 파악 → 코드 수정 후 재처리
```

> 💡 **DLQ (Dead Letter Queue, 데드 레터 큐)** = **"처리 실패한 메시지를 버리지 않고 모아두는 격리 병실"**.
> DLQ가 없으면 실패 메시지는 그냥 **사라집니다**. "주문은 됐는데 재고가 안 깎였다"는 사고가 나도 **증거가 안 남습니다.** DLQ는 EDA 운영의 필수품입니다.

> ⚠️ `max-attempts` 재시도는 **애플리케이션 메모리 안에서 짧은 간격(기본 1초→2초...)으로 반복**됩니다. 그래서 **DB 커넥션 일시 고갈 같은 순간적 장애에는 유효하지만, 외부 API가 10분간 죽은 상황에는 무의미**합니다. 후자는 지연 재시도 큐 같은 별도 설계가 필요합니다.

## 1-6. 전달 보장 수준과 멱등성 ★반드시 알아야 함★

메시지 시스템은 세 가지 보장 수준이 있습니다.

| 수준 | 의미 | 특징 |
|---|---|---|
| **At-most-once** (최대 1번) | 유실될 수 있지만 중복은 없음 | 빠름. 로그/통계용 |
| **At-least-once** (최소 1번) ★ | **중복될 수 있지만 유실은 없음** | **RabbitMQ + Spring Cloud Stream 기본값** |
| **Exactly-once** (정확히 1번) | 유실도 중복도 없음 | 분산 시스템에서 **사실상 불가능**하거나 매우 비쌈 |

**우리 프로젝트는 At-least-once입니다.** 즉 **같은 메시지가 2번 이상 도착할 수 있습니다.**

왜 중복이 생기나?

- 컨슈머가 처리는 다 했는데 ACK를 보내기 직전에 **프로세스가 죽으면** → 브로커는 "안 받았네" 하고 재전송합니다.
- 네트워크가 끊겨 ACK가 유실되어도 마찬가지입니다.

그래서 EDA에서 컨슈머는 **멱등(Idempotent)** 해야 합니다.

> **멱등성(Idempotency)** = 같은 요청을 **몇 번 실행해도 결과가 한 번 실행한 것과 같은** 성질.
> - ❌ 멱등하지 않음: `재고 = 재고 - 2` (두 번 실행하면 4개가 깎임)
> - ✅ 멱등함: `주문 상태 = COMPLETED` (몇 번 해도 COMPLETED)

`eda-project2`의 `Order` 엔티티는 이 점을 의식하고 있습니다.

```java
public void complete() {
    if (this.status == OrderStatus.PENDING) {   // ★ PENDING일 때만 바꾼다
        this.status = OrderStatus.COMPLETED;
    }
}
```

상태가 이미 `COMPLETED`인데 같은 이벤트가 또 와도 아무 일도 일어나지 않습니다 → **멱등**. (반면 재고 차감은 멱등하지 않습니다 → Part 7에서 다룹니다)

---

# Part 2. Spring Cloud Stream 이해하기

## 2-1. 왜 이런 게 필요한가

RabbitMQ를 스프링에서 그냥 쓰려면 (`spring-boot-starter-amqp`) 이런 코드를 직접 써야 합니다.

```java
@Bean Queue queue() { return new Queue("order-created-topic.stock-group", true); }
@Bean TopicExchange exchange() { return new TopicExchange("order-created-topic"); }
@Bean Binding binding(Queue q, TopicExchange e) { return BindingBuilder.bind(q).to(e).with("order.*"); }

@RabbitListener(queues = "order-created-topic.stock-group")
public void handle(OrderCreatedEvent event) { ... }

rabbitTemplate.convertAndSend("order-created-topic", "order.created", event);
```

문제는 이 코드가 **RabbitMQ 전용**이라는 점입니다. Kafka로 바꾸려면 **비즈니스 로직 클래스까지 전부 다시** 써야 합니다.

**Spring Cloud Stream(SCSt)** 은 이 문제를 **브로커 추상화**로 해결합니다.

## 2-2. 3계층 구조 — SCSt의 심장

```
┌───────────────────────────────────────────────────────────────┐
│  ① 애플리케이션 코드                                            │
│     Consumer<OrderCreatedEvent> / StreamBridge.send(...)       │
│     → 브로커를 전혀 모릅니다. 그냥 자바 함수입니다.               │
└──────────────────────────┬────────────────────────────────────┘
                           │  논리적 채널 이름으로 연결
                           │  ("stockControl-in-0", "orderCreated-out-0")
┌──────────────────────────▼────────────────────────────────────┐
│  ② Binding (바인딩) — application.yml의 설정                    │
│     "이 채널은 order-created-topic 이라는 목적지에 연결"          │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│  ③ Binder (바인더) — 의존성으로 갈아끼우는 어댑터                 │
│     spring-cloud-stream-binder-rabbit  ← 우리가 쓰는 것          │
│     spring-cloud-stream-binder-kafka   ← 이걸로 바꾸면 카프카      │
│     "목적지(destination)"를 브로커의 실제 물건으로 번역            │
└──────────────────────────┬────────────────────────────────────┘
                           ▼
                  RabbitMQ (Exchange/Queue/Binding 자동 생성)
```

**핵심 원리**: 애플리케이션은 **"논리적 목적지(destination)"** 라는 추상적 이름만 압니다. 그것이 RabbitMQ의 **Exchange**인지, Kafka의 **Topic**인지는 **바인더가 번역**합니다.

| SCSt 개념 | RabbitMQ에서는 | Kafka에서는 |
|---|---|---|
| `destination` | **Exchange** | Topic |
| `group` | `destination.group` 이름의 **Queue** | Consumer Group |
| 파티셔닝 | 라우팅 키로 흉내 | 네이티브 파티션 |

> 💡 이 프로젝트에서 **여러분이 RabbitMQ 큐를 하나도 직접 만들지 않았다**는 점을 주목하세요. 애플리케이션이 뜰 때 **바인더가 설정을 읽고 Exchange, Queue, Binding, DLQ를 전부 자동으로 만듭니다(auto-provisioning).** 관리 콘솔(http://localhost:15672)에서 확인할 수 있습니다.

## 2-3. 함수형 프로그래밍 모델 ★

옛날 SCSt는 `@EnableBinding`, `@StreamListener` 같은 애노테이션을 썼지만 **지금은 전부 폐기(deprecated)** 되었습니다. 현재 모델은 **"자바 표준 함수 인터페이스를 빈으로 등록하면 그게 메시지 핸들러다"** 입니다.

| 자바 함수 인터페이스 | 역할 | 채널 | 비유 |
|---|---|---|---|
| `Supplier<O>` | 입력 없이 **출력만** | out만 생성 | 발행 전용 (주기적 발행) |
| `Function<I, O>` | 입력받아 **가공 후 출력** | in + out | 파이프라인 중간 단계 |
| **`Consumer<I>`** ★ | 입력만 받고 **출력 없음** | in만 생성 | **구독 전용** (이 프로젝트가 쓰는 것) |

```java
@Configuration
public class StockConsumer {
    @Bean                                        // ★ 빈 이름 = "stockControl"
    public Consumer<OrderCreatedEvent> stockControl() {
        return event -> { /* 여기가 메시지 수신 지점 */ };
    }
}
```

> 💡 **왜 클래스에 `@Configuration`이 붙었나?** 이 클래스는 컨트롤러가 아니라 **"stockControl이라는 이름의 함수 빈을 등록하는 설정 클래스"** 이기 때문입니다. 메서드 이름 `stockControl`이 곧 **빈 이름이자 채널 이름의 접두어**가 됩니다. **이름 하나로 코드와 YAML이 연결**되므로, 메서드 이름을 바꾸면 YAML도 같이 바꿔야 합니다.

## 2-4. 채널 이름 규칙 ★암기 필수★

```
<함수이름>-in-<인덱스>     ← 입력(수신) 채널
<채널이름>-out-<인덱스>    ← 출력(발행) 채널

인덱스는 0부터. 대부분 0입니다. (입출력이 여러 개인 고급 함수에서만 1, 2...)
```

| 프로젝트의 실제 채널 | 방향 | 의미 |
|---|---|---|
| `stockControl-in-0` | 수신 | `stockControl` 함수 빈의 0번 입력 |
| `stockSuccess-in-0` | 수신 | `stockSuccess` 함수 빈의 0번 입력 |
| `orderCreated-out-0` | 발행 | `StreamBridge`가 쓸 발행 채널 |
| `stockReserved-out-0` | 발행 | 재고 성공 이벤트 발행 채널 |

> ⚠️ **가장 흔한 실수**: 함수 이름과 `-in-0` 앞의 이름이 다르면 **아무 일도 안 일어납니다** (에러도 안 남). 메시지가 안 온다면 여기부터 확인하세요.

## 2-5. `spring.cloud.function.definition` — 활성화 스위치

빈으로 등록만 하면 SCSt가 알아서 다 잡아주지 않습니다. **"이 함수들을 메시지 핸들러로 쓰겠다"** 고 명시해야 합니다.

```yaml
spring:
  cloud:
    function:
      definition: stockSuccess;stockFailure   # ★ 세미콜론(;)으로 여러 개 나열
```

| 구분자 | 의미 |
|---|---|
| `;` (세미콜론) | **독립적인 여러 함수**를 각각 활성화 — 이 프로젝트가 쓰는 것 |
| `\|` (파이프) | **함수 합성** — `a\|b` 는 a의 출력이 b의 입력으로 (하나의 파이프라인) |

> 💡 발행 전용(`StreamBridge`만 쓰는) 경우는 `definition`에 안 적어도 됩니다. `eda-project`의 order-service에 `function.definition`이 없는 이유입니다.

## 2-6. StreamBridge — "원할 때 발행하기"

`Supplier`는 **스프링이 주기적으로 호출**(기본 1초마다)하는 구조라, "주문이 저장된 **바로 그 순간**에 발행"에는 안 맞습니다.

**`StreamBridge`** 는 **비즈니스 로직 한가운데서 명령형으로 이벤트를 밀어 넣는 도구**입니다.

```java
streamBridge.send("orderCreated-out-0", event);
//                 ↑ 채널 이름          ↑ 페이로드(자바 객체)
```

이 한 줄 안에서 벌어지는 일:

```
1. 채널 이름으로 바인딩 설정 조회       → destination = "order-created-topic"
2. 자바 객체 → 메시지 변환 (직렬화)
      Jackson이 record를 JSON 문자열로 변환
      {"orderId":"...","productId":"apple","quantity":2,"occurredAt":"..."}
      헤더 contentType: application/json 자동 부착
3. 라우팅 키 결정 (routing-key-expression 설정 → "order.created")
4. RabbitMQ의 order-created-topic Exchange로 publish
5. Exchange가 바인딩 규칙에 따라 큐로 분배
```

> 💡 `send()`의 반환값은 `boolean`이지만, 이것은 **"채널로 보냈다"** 는 뜻이지 **"브로커가 디스크에 저장했다"** 는 보장이 아닙니다. 진짜 보장을 원하면 **Publisher Confirms** 설정이 필요합니다 (이 실습에는 없음 → Part 7).

## 2-7. `bindings` vs `rabbit.bindings` — 두 층으로 나뉜 설정

YAML을 처음 보면 `bindings`가 두 번 나와서 헷갈립니다. 구조는 이렇습니다.

```yaml
spring:
  cloud:
    stream:
      bindings:                    # ← ① 브로커 중립적인 "공통" 설정
        stockControl-in-0:
          destination: ...         #   (Kafka로 바꿔도 그대로 유효)
          group: ...
          consumer:
            max-attempts: 3
      rabbit:                      # ← ② RabbitMQ에만 있는 "전용" 설정
        bindings:
          stockControl-in-0:
            consumer:
              binding-routing-key: 'order.*'   # (Kafka엔 이런 개념이 없음)
              auto-bind-dlq: true
```

| 위치 | 성격 | 예 |
|---|---|---|
| `spring.cloud.stream.bindings` | **표준/추상** — 브로커가 바뀌어도 유지 | `destination`, `group`, `content-type`, `max-attempts` |
| `spring.cloud.stream.rabbit.bindings` | **RabbitMQ 전용** — 브로커 바꾸면 버려짐 | `routing-key-expression`, `binding-routing-key`, `auto-bind-dlq`, `republish-to-dlq` |

> 💡 이 분리 자체가 Part 2-2에서 말한 **추상화 계층의 물리적 증거**입니다. 위 칸은 "무엇을(what)", 아래 칸은 "RabbitMQ에서 어떻게(how)".

## 2-8. 설정 → RabbitMQ 실물 매핑표 (치트시트)

| YAML 설정 | RabbitMQ에 실제로 만들어지는 것 |
|---|---|
| `destination: order-created-topic` | **Topic Exchange** `order-created-topic` |
| `group: stock-group` | **Queue** `order-created-topic.stock-group` (durable) |
| `group` 없음 | 익명 임시 Queue `order-created-topic.anonymous.xxxx` (auto-delete) |
| `binding-routing-key: 'order.*'` | 위 Exchange↔Queue 사이의 **Binding**, 키 `order.*` |
| `binding-routing-key` 생략 | Binding 키 `#` (전부 수신) |
| `routing-key-expression: "'order.created'"` | 발행 메시지에 붙는 **라우팅 키** |
| `routing-key-expression` 생략 | 라우팅 키 = destination 이름 그대로 |
| `auto-bind-dlq: true` | **DLX**(Dead Letter Exchange) + **DLQ** `order-created-topic.stock-group.dlq` |

> ⚠️ `routing-key-expression`의 값이 `"'order.created'"` 처럼 **따옴표가 이중**인 이유: 이 설정은 **SpEL(Spring Expression Language) 표현식**입니다. 따옴표를 한 겹만 쓰면 SpEL이 `order.created`를 **"변수/프로퍼티 경로"로 해석**해서 실패합니다. 안쪽 작은따옴표가 **"이건 문자열 리터럴이다"** 를 뜻합니다.
> (동적으로 만들고 싶다면 `headers['type']` 나 `payload.productId` 같이 쓸 수 있습니다.)

---

# Part 3. 두 프로젝트의 공통 뼈대

## 3-1. Gradle 멀티모듈 구조

```
eda-project/
├── settings.gradle          ← 어떤 모듈이 이 프로젝트에 속하는지 선언
├── build.gradle             ← 모든 모듈에 공통 적용되는 설정 (부모)
├── docker-compose.yml       ← 인프라(RabbitMQ, MySQL)
├── common-module/           ← 서비스들이 공유하는 DTO/이벤트 클래스
├── order-service/           ← 주문 마이크로서비스 (포트 8081)
└── stock-service/           ← 재고 마이크로서비스 (포트 8082)
```

```groovy
// settings.gradle
rootProject.name = 'eda-project'
include 'common-module'
include 'order-service'
include 'stock-service'
```

**루트 `build.gradle`의 원리**:

```groovy
plugins {
  id 'org.springframework.boot' version '4.1.1' apply false  // ★ apply false
}
subprojects { ... }   // 하위 모듈 전체에 공통 적용
```

- `apply false` = **"플러그인 버전은 여기서 정하되, 루트 프로젝트 자신에게는 적용하지 마라"**. 루트는 실행 가능한 앱이 아니라 **껍데기**이기 때문입니다.
- `subprojects { }` 블록 = **모든 하위 모듈에 Java 21, Lombok, 스프링 부트 BOM을 한 번에 상속**시킵니다. 버전을 한 곳에서 관리하는 효과.
- `mavenBom(...)` = **의존성 버전 카탈로그**. 개별 라이브러리에 버전을 안 적어도 되는 이유입니다 (`implementation 'org.springframework.cloud:spring-cloud-stream'`에 버전이 없죠).

## 3-2. `common-module` — 왜 필요하고, 왜 저 설정인가

주문 서비스가 발행하는 `OrderCreatedEvent`의 필드 이름/타입과, 재고 서비스가 수신하는 그것이 **정확히 같아야** JSON 역직렬화가 됩니다. 이 **계약(Contract)** 을 공유하는 모듈이 `common-module`입니다.

```groovy
// common-module/build.gradle
bootJar { enabled = false }   // ★
jar     { enabled = true }    // ★
```

**왜 이 설정이 필요한가 (중요한 원리)**:

- `bootJar`는 스프링 부트가 만드는 **실행 가능한 뚱뚱한 JAR**입니다. 내부 구조가 `BOOT-INF/classes/...` 로 재배치되어 있어서 **다른 프로젝트가 라이브러리로 가져다 쓸 수 없습니다.**
- `common-module`은 실행할 `main()` 이 없는 **순수 라이브러리**입니다. 그러므로 부트 JAR 생성을 끄고(`bootJar false`) **평범한 일반 JAR**(`jar true`)로 만들어야 `order-service`가 `implementation project(':common-module')` 로 참조할 수 있습니다.
- 이 설정을 빠뜨리면 클래스를 못 찾는 문제가 생깁니다. 멀티모듈 스프링 프로젝트의 단골 실수입니다.

> ⚠️ **설계 관점의 주의**: 공용 DTO 모듈은 편하지만, **모든 서비스를 하나의 클래스에 다시 결합시키는 양날의 검**입니다. `OrderCreatedEvent`에 필드를 하나 추가하면 모든 서비스를 다시 빌드/배포해야 할 수 있습니다. 실무에서는 (a) 필드 추가만 허용하고 삭제/변경 금지, (b) 스키마 레지스트리 사용, (c) 각 서비스가 자기 DTO를 복사해서 갖기 — 중 하나를 택합니다. **학습용으로는 공유 모듈이 압도적으로 편하므로 이 프로젝트는 편의상 공유를 택했습니다.**

## 3-3. `record`를 쓰는 이유

```java
public record OrderCreatedEvent(
    String orderId, String productId, Integer quantity, Instant occurredAt) {}
```

`record`는 Java 16+의 **불변(immutable) 데이터 전용 클래스**입니다. 생성자, 접근자(`orderId()`), `equals`, `hashCode`, `toString`이 자동 생성됩니다.

**이벤트에 record가 완벽하게 어울리는 이유**: Part 0-3에서 말했듯 이벤트는 **"이미 일어난, 변경 불가능한 사실"** 입니다. 값을 바꿀 수 없는 `record`는 이 의미를 **타입 시스템으로 강제**합니다. `event.setQuantity(5)` 같은 코드가 **애초에 존재할 수 없습니다.**

`occurredAt` 필드도 이벤트 설계의 관례입니다 — **"언제 일어난 사실인가"** 는 이벤트의 필수 메타데이터입니다 (순서 판단, 지연 측정, 감사 로그에 사용).

## 3-4. Java 21 가상 스레드

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

**배경 지식**: 전통적인 자바 스레드(플랫폼 스레드)는 **OS 스레드와 1:1**로 묶여 있어 개당 ~1MB 스택을 먹습니다. 그래서 톰캣 기본 스레드풀은 200개 정도가 한계입니다. 그리고 스레드가 **DB 응답을 기다리며 블로킹**되면, 그 비싼 OS 스레드가 **아무 일도 안 하면서 점유**됩니다.

**가상 스레드(Virtual Thread, Project Loom)** 는 JVM이 관리하는 **아주 가벼운 스레드**입니다. 핵심 원리는 **마운트/언마운트**입니다.

```
가상 스레드가 DB I/O로 블로킹되는 순간
   → JVM이 그 가상 스레드를 OS 스레드에서 "떼어냅니다"(unmount)
   → 비워진 OS 스레드는 즉시 다른 가상 스레드를 실행합니다
   → I/O 응답이 오면 다시 아무 OS 스레드에 "붙입니다"(mount)
```

결과적으로 **동기식 블로킹 코드(읽기 쉬움)를 그대로 쓰면서, 비동기의 처리량**을 얻습니다. 리액티브(WebFlux)의 어려운 문법 없이요.

**이 프로젝트에 왜 켰나**: 메시지 컨슈머는 본질적으로 **"메시지 받고 → DB 접근하고(블로킹) → 끝"** 의 반복입니다. 정확히 가상 스레드가 이득 보는 패턴입니다. 켜면 스프링 부트가 톰캣과 **RabbitMQ 리스너 컨테이너의 실행자(Executor)** 를 가상 스레드 기반으로 교체합니다.

> ⚠️ 주의점: 가상 스레드는 **CPU를 태우는 계산 작업에는 이득이 없습니다** (OS 스레드 수만큼만 병렬). 또 `synchronized` 블록 안에서 블로킹하면 **핀(pinning)** 이 발생해 이점이 사라질 수 있습니다(최신 JDK에서 대부분 해소). `eda-project`의 `Thread.sleep(150)`은 가상 스레드에서 **언마운트되는** 대표적 예라 학습 예제로 적절합니다.
---

# Part 4. eda-project #1 — 이벤트 발행과 구독의 기본기

## 4-0. 이 프로젝트의 학습 목표

> **"주문 서비스가 재고 서비스를 전혀 모르는 채로, 재고가 깎이게 만들기"**

DB도 없습니다(로그로 대체). 오직 **이벤트가 흘러가는 파이프라인 자체**에만 집중합니다.

## 4-1. 전체 그림

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │ order-service (:8081)                                                  │
 │                                                                        │
 │  ① POST /api/orders/v1                                                 │
 │        │                                                               │
 │        ▼                                                               │
 │  ② OrderController ──▶ OrderService.createOrderV1()  @Transactional     │
 │        │                    ├ UUID로 orderId 생성                       │
 │        │                    ├ (DB 저장 — 로그로 가정)                    │
 │        │                    └ eventPublisher.publishEvent(event)  ③     │
 │        │                          ↑ 아직 RabbitMQ 아님! 애플리케이션 내부  │
 │        ▼                                                               │
 │  ④ 트랜잭션 커밋 완료                                                    │
 │        │                                                               │
 │        ▼                                                               │
 │  ⑤ OrderEventRelay @TransactionalEventListener(AFTER_COMMIT)           │
 │        └ streamBridge.send("orderCreated-out-0", event)                │
 └────────────────────────────┬───────────────────────────────────────────┘
                              │ JSON + routing key "order.created"
                              ▼
        ┌─────────────────────────────────────────────────────┐
        │ RabbitMQ                                             │
        │  Exchange: order-created-topic (topic)               │
        │       │ binding key: order.*                         │
        │       ▼                                              │
        │  Queue: order-created-topic.stock-group              │
        │       │                       ↘ (3회 실패 시)         │
        │       │                    Queue: ....stock-group.dlq│
        └───────┼──────────────────────────────────────────────┘
                ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │ stock-service (:8082)                                                  │
 │  ⑥ Consumer<OrderCreatedEvent> stockControl                            │
 │        └ 재고 차감 (Thread.sleep(150)으로 가정) → 로그                    │
 └────────────────────────────────────────────────────────────────────────┘
```

## 4-2. 코드 정독 — order-service

### (1) `OrderController` — 평범한 REST 진입점

```java
@RestController
@RequiredArgsConstructor            // Lombok: final 필드를 받는 생성자 자동 생성 → 생성자 주입
@RequestMapping("/api/orders")
public class OrderController {
  private final OrderService orderService;

  @PostMapping("/v1")
  public ResponseEntity<String> createOrderV1(@RequestBody OrderCreatedRequest request) {
    String orderId = orderService.createOrderV1(request);
    return ResponseEntity.ok("[v1] Order Created: " + orderId);
  }
}
```

**여기서 이해할 점**: 이 컨트롤러는 **재고에 대해 한 글자도 언급하지 않습니다.** 그런데도 재고가 깎입니다. 이것이 EDA입니다. 그리고 응답은 **재고 처리를 기다리지 않고 즉시** 반환됩니다.

### (2) `OrderService` — 이벤트를 "밖"이 아니라 "안"으로 먼저 던진다 ★핵심★

```java
@Transactional
public String createOrderV1(OrderCreatedRequest request) {
    String orderId = UUID.randomUUID().toString();
    log.info("[Order Saved] ID: {}", orderId);          // DB Insert 가정

    OrderCreatedEvent event = new OrderCreatedEvent(
        orderId, request.productId(), request.quantity(), Instant.now());

    eventPublisher.publishEvent(event);   // ★ RabbitMQ가 아니라 "스프링 내부"로 발행
    return orderId;
}
```

여기서 가장 헷갈리는 부분을 짚습니다.

> **Q. `publishEvent()`가 RabbitMQ로 보내는 건가요?**
> **A. 아닙니다.** `ApplicationEventPublisher`는 **스프링 컨테이너 안에서만 도는 인메모리 이벤트 버스**입니다. 네트워크를 타지 않습니다. 같은 JVM 안의 리스너에게만 전달됩니다.

**그럼 왜 굳이 두 단계(내부 이벤트 → 외부 이벤트)로 나눴을까요?** 이것이 이 프로젝트의 가장 중요한 설계 의도입니다.

```java
// ❌ 만약 서비스 안에서 바로 RabbitMQ로 보냈다면?
@Transactional
public String createOrderBad(...) {
    orderRepository.save(order);
    streamBridge.send("orderCreated-out-0", event);  // ← 아직 커밋 전!
    somethingElse();                                  // ← 여기서 예외가 난다면?
}
```

이 코드의 문제:

1. `save()`는 아직 **커밋되지 않았습니다.** 트랜잭션이 끝나야 DB에 확정됩니다.
2. 그런데 이벤트는 **이미 RabbitMQ로 나가버렸습니다.**
3. `somethingElse()`에서 예외 → **주문은 롤백되어 사라졌는데, 재고는 이미 깎였습니다.**

이런 **"존재하지 않는 주문에 대한 이벤트"** 를 **유령 이벤트(Phantom Event)** 라고 합니다. 이벤트는 "일어난 사실"이어야 하는데, 일어나지 않은 일을 알린 셈입니다. 되돌릴 방법도 없습니다(이미 브로커로 나갔으므로).

**해결책이 바로 다음 (3)번입니다.**

### (3) `OrderEventRelay` — `@TransactionalEventListener(AFTER_COMMIT)` ★가장 중요한 개념★

```java
@Component
public class OrderEventRelay {
  private final StreamBridge streamBridge;

  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)   // ★
  public void handleOrderCommitted(OrderCreatedEvent event) {
    streamBridge.send("orderCreated-out-0", event);
    log.info("[Event Published to RabbitMQ] Payload: {}", event);
  }
}
```

**동작 원리 — 시간 순서로 보기**:

```
시각 t1  createOrderV1() 시작 (트랜잭션 시작)
시각 t2  orderRepository.save()
시각 t3  publishEvent(event)
           → 스프링이 이벤트를 즉시 배달하지 않고
             "현재 트랜잭션에 딸린 대기 목록"에 보관해 둠  ★
시각 t4  createOrderV1() 정상 종료 → 트랜잭션 COMMIT
시각 t5  ★ 커밋이 물리적으로 끝난 직후 ★
           스프링이 대기 목록을 꺼내 handleOrderCommitted() 실행
           → 이제서야 RabbitMQ로 발행
```

만약 t4에서 **롤백**되면? 대기 목록은 **그냥 버려집니다.** → **유령 이벤트가 원천 차단됩니다.**

**`phase` 옵션 네 가지**:

| phase | 실행 시점 | 용도 |
|---|---|---|
| `BEFORE_COMMIT` | 커밋 직전 | 같은 트랜잭션에 뭔가 더 쓰고 싶을 때 |
| **`AFTER_COMMIT`** (기본값) ★ | **커밋 성공 직후** | **외부 시스템 통지 — 이 프로젝트** |
| `AFTER_ROLLBACK` | 롤백 직후 | 실패 알림/보상 |
| `AFTER_COMPLETION` | 커밋/롤백 무관하게 끝난 후 | 리소스 정리 |

> 💡 **왜 별도 클래스(`OrderEventRelay`)로 분리했나?**
> **책임 분리**입니다. `OrderService`는 "주문 업무"만 알고, "메시징 인프라"는 몰라야 합니다. 이 릴레이 클래스가 **도메인과 인프라 사이의 번역기(Anti-Corruption Layer)** 역할을 합니다. 나중에 RabbitMQ를 Kafka로 바꿔도 **`OrderService`는 손도 안 댑니다.**

> ⚠️ **1번 프로젝트에서의 주의**: `eda-project`에는 DataSource/JPA 의존성이 없어서 **트랜잭션 매니저 빈이 존재하지 않습니다.** 스프링 부트는 트랜잭션 매니저가 없으면 트랜잭션 기능 자체를 활성화하지 않으므로, 여기서의 `@Transactional`과 `AFTER_COMMIT`은 **"기다릴 커밋이 실제로는 없는"** 상태입니다. 즉 **구조를 미리 갖춰 둔 형태**에 가깝고, 이 애노테이션의 진짜 가치는 **JPA+MySQL이 붙은 `eda-project2`에서 비로소 발휘**됩니다.
> 직접 확인해보고 싶다면 `eda-project`의 order-service에 H2 같은 DataSource를 추가한 뒤, 서비스 마지막 줄에 일부러 예외를 던져 보세요. 이벤트가 **발행되지 않는 것**을 볼 수 있습니다.

### (4) 발행 측 YAML

```yaml
spring:
  cloud:
    stream:
      bindings:
        orderCreated-out-0:                 # StreamBridge가 쓰는 채널 이름과 일치해야 함
          destination: order-created-topic  # → Topic Exchange 이름
          content-type: application/json
      rabbit:
        bindings:
          orderCreated-out-0:
            producer:
              routing-key-expression: "'order.created'"   # 메시지에 붙일 주소 라벨
```

**이 설정이 없다면?** 라우팅 키가 destination 이름(`order-created-topic`)이 되어버려서, 컨슈머 쪽의 `order.*` 패턴에 **매칭되지 않아 메시지가 사라집니다.** (Exchange는 갈 곳이 없는 메시지를 조용히 버립니다 — 에러도 안 납니다. EDA 디버깅이 어려운 대표적 이유입니다.)

## 4-3. 코드 정독 — stock-service

### (1) `StockConsumer` — 함수 하나가 곧 리스너

```java
@Configuration
public class StockConsumer {
  @Bean
  public Consumer<OrderCreatedEvent> stockControl() {
    return event -> {
      log.info("[Event Received] Order: {}, ...", event.orderId(), ...);
      try {
        Thread.sleep(150);                        // 재고 처리(I/O)를 가정
        log.info("[Stock Successfully Deducted] Product: {}", event.productId());
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();        // ★ 인터럽트 상태 복원
        throw new IllegalStateException("재고 처리 중 스레드 인터럽트 발생", e);
      }
    };
  }
}
```

**뜯어볼 점 세 가지**:

1. **`@RabbitListener`도, 큐 이름도 없습니다.** 함수 이름 `stockControl` + YAML의 `stockControl-in-0` 이 둘만으로 연결됩니다.
2. **역직렬화(JSON → 자바 객체)를 아무도 안 했습니다.** `Consumer<OrderCreatedEvent>`의 **제네릭 타입**을 SCSt가 읽어서 Jackson으로 자동 변환합니다. 이래서 `common-module`로 타입을 공유하는 것입니다.
3. **`Thread.currentThread().interrupt()`** — `InterruptedException`을 잡으면 JVM이 인터럽트 플래그를 꺼버립니다. 그대로 두면 상위 코드가 "중단 요청이 있었다"는 사실을 모릅니다. 그래서 플래그를 **복원**하고 런타임 예외로 승격시키는 것이 관례입니다. 여기서 예외를 던지면 → **재시도 3회 → DLQ**로 흘러갑니다.

### (2) 수신 측 YAML — 한 줄씩

```yaml
spring:
  cloud:
    function:
      definition: stockControl          # ① 이 함수를 메시지 핸들러로 활성화
    stream:
      bindings:
        stockControl-in-0:              # ② <함수명>-in-<인덱스>
          destination: order-created-topic   # ③ 구독할 Exchange (발행측과 반드시 동일)
          group: stock-group                 # ④ 물리 큐 생성 + 경쟁 소비자
          consumer:
            max-attempts: 3                  # ⑤ 실패 시 총 3회 시도
      rabbit:
        bindings:
          stockControl-in-0:
            consumer:
              binding-routing-key: 'order.*' # ⑥ 이 패턴의 메시지만 큐로 받음
              auto-bind-dlq: true            # ⑦ DLQ 자동 생성 + 연결
              republish-to-dlq: true         # ⑧ 에러 정보를 헤더에 담아 DLQ로
```

이 8줄이 만들어내는 RabbitMQ의 실제 모습:

| 번호 | 만들어지는 것 |
|---|---|
| ③ | Topic Exchange `order-created-topic` |
| ④ | Queue `order-created-topic.stock-group` (durable) |
| ⑥ | Binding: 위 Exchange → 위 Queue, key `order.*` |
| ⑦ | Exchange `DLX` + Queue `order-created-topic.stock-group.dlq` |

## 4-4. 직접 실행하고 확인하기

```bash
# 1) 인프라 기동
cd eda-project
docker compose up -d

# 2) 두 서비스 기동 (터미널 2개)
./gradlew :order-service:bootRun
./gradlew :stock-service:bootRun

# 3) 주문 요청
curl -X POST http://localhost:8081/api/orders/v1 \
  -H "Content-Type: application/json" \
  -d '{"productId":"apple","quantity":2}'
```

**기대 로그**:

```
[order-service] [Order Saved] ID: 3f2b...
[order-service] [Event Published to RabbitMQ] Payload: OrderCreatedEvent[...]
[stock-service] [Event Received] Order: 3f2b..., Product: apple, Quantity: 2
[stock-service] [Stock Successfully Deducted] Product: apple
```

**꼭 해볼 실험 3가지**:

| 실험 | 방법 | 배우는 것 |
|---|---|---|
| **시간적 분리** | stock-service를 **끄고** 주문 3건 요청 → 그 다음 stock-service 켜기 | 주문은 성공하고, 켜는 순간 3건이 몰아서 처리됨. **`group` 덕분에 큐에 보관됨** |
| **DLQ 동작** | `stockControl`에서 무조건 `throw new RuntimeException()` | 로그에 3번 시도 후, 관리 콘솔의 `...stock-group.dlq`에 메시지 1건이 쌓임 |
| **라우팅 키 매칭** | `routing-key-expression`을 `"'payment.done'"` 으로 변경 | `order.*`에 안 걸려서 **메시지가 조용히 증발**. Exchange의 라우팅 원리 체감 |

**관리 콘솔**: http://localhost:15672 (guest / guest) → Exchanges, Queues 탭에서 자동 생성된 객체들을 눈으로 확인하세요.

---

# Part 5. eda-project2 — SAGA 패턴으로 분산 트랜잭션 다루기

## 5-0. 1번 프로젝트가 못 푼 문제

`eda-project`는 **"주문이 생겼다"는 사실을 알리는 것까지**만 합니다. 그런데 현실에서 이런 일이 벌어집니다.

> 주문은 접수됐는데 → **재고가 부족해서 차감에 실패했다.**
> 그런데 손님은 이미 "주문 완료" 응답을 받았다. **주문 데이터는 DB에 남아 있다.**

1번 프로젝트에는 이 상황을 처리할 방법이 **아예 없습니다.** 재고 서비스가 실패해도 주문 서비스는 그 사실을 **영원히 모릅니다.**

## 5-1. 왜 분산 트랜잭션은 어려운가

DB 하나면 `@Transactional`이 전부 해결합니다. DB가 둘이면?

**옛날 정답: 2PC (Two-Phase Commit, 2단계 커밋)**

```
조정자: "다들 커밋 준비됐어?"  ← 1단계 (Prepare)
  주문DB: "준비 완료"
  재고DB: "준비 완료"
조정자: "그럼 다같이 커밋!"     ← 2단계 (Commit)
```

**MSA에서 2PC를 안 쓰는 이유**:

| 문제 | 설명 |
|---|---|
| **잠금 시간이 김** | 1단계와 2단계 사이 내내 **양쪽 DB의 락이 잡혀 있습니다.** 처리량이 급락합니다. |
| **조정자가 단일 장애점** | 조정자가 2단계 직전에 죽으면 **모든 참여자가 락을 쥔 채 영원히 대기**합니다. |
| **참여 불가한 것들** | REST API, 메시지 브로커, 외부 결제사 등은 **애초에 2PC에 참여할 수 없습니다.** |
| **가용성 파괴** | 한 곳이 느려지면 전부 느려집니다 — MSA로 쪼갠 이유가 사라집니다. |

## 5-2. SAGA 패턴 ★핵심 개념★

> **SAGA** = 하나의 큰 분산 트랜잭션을, **각자 로컬 트랜잭션으로 즉시 커밋하는 여러 단계**로 쪼개고, 중간에 실패하면 **"취소"가 아니라 "되돌리는 새 작업"** 을 실행하는 패턴.

핵심은 **롤백(rollback)이 아니라 보상(compensation)** 입니다.

```
전통적 트랜잭션 (롤백)          SAGA (보상 트랜잭션)
─────────────────────          ──────────────────────────
① 주문 생성  ┐                 ① 주문 생성 → 즉시 커밋 ✅ (되돌릴 수 없음)
② 재고 차감  ├ 통째로 커밋       ② 재고 차감 → 실패 ❌
③ 결제      ┘ 또는 롤백         ③ ★보상★: "주문 취소" 라는 새 작업을 실행
                                     → 상태를 CANCELLED로 변경
```

> 💡 **은행 비유**: 이미 인쇄된 잘못된 수표는 **없던 일로 만들 수 없습니다**. 대신 **"취소 전표"라는 새 기록을 남깁니다**. 회계에서 이걸 역분개라고 하죠. SAGA의 보상 트랜잭션이 정확히 이 개념입니다. **흔적이 남습니다.** 그리고 이 흔적이 오히려 감사/추적에 유리합니다.

### SAGA의 두 가지 구현 방식

| | **코레오그래피 (Choreography)** ★이 프로젝트★ | **오케스트레이션 (Orchestration)** |
|---|---|---|
| 비유 | **군무(群舞)** — 지휘자 없이 각자 음악 듣고 춤춤 | **오케스트라** — 지휘자가 순서를 지시 |
| 구조 | 각 서비스가 이벤트를 듣고 **스스로 다음 행동 결정** | 중앙 **오케스트레이터**가 각 서비스에 명령 |
| 장점 | 단순, 중앙 장애점 없음, 서비스 추가가 쉬움 | 흐름이 **한 곳에 보임**, 디버깅 쉬움 |
| 단점 | 흐름이 코드 어디에도 안 보임, 단계가 늘면 **순환 참조** 위험 | 오케스트레이터가 커지고 단일 장애점이 됨 |
| 적합 | **참여 서비스 2~4개** | 참여 서비스 5개 이상, 복잡한 분기 |

`eda-project2`는 **코레오그래피**입니다. 지휘자가 없고, **이벤트가 이벤트를 부르는 연쇄**로 흐름이 만들어집니다.

## 5-3. 전체 이벤트 흐름

### 성공 시나리오

```
 order-service                RabbitMQ                    stock-service
 ─────────────                ─────────                   ──────────────
 POST /api/orders/v1
    │
    ├ Order 저장 (status=PENDING) ── 로컬 트랜잭션 ① 커밋
    │
    └ OrderCreatedEvent ────▶ order-created-topic ────────▶ stockControl
                              (rk: order.created)              │
                                                               ├ 비관적 락으로 재고 조회
                                                               ├ 재고 차감 ── 로컬 트랜잭션 ② 커밋
                                                               │
    stockSuccess ◀────────── stock-success-topic ◀────────────┘ StockReservedEvent
        │
        └ order.complete() → status = COMPLETED ── 로컬 트랜잭션 ③ 커밋
```

### 실패 시나리오 (재고 부족)

```
 order-service                RabbitMQ                    stock-service
 ─────────────                ─────────                   ──────────────
 POST /api/orders/v1
    │
    ├ Order 저장 (status=PENDING) ── 로컬 트랜잭션 ① 커밋
    │
    └ OrderCreatedEvent ────▶ order-created-topic ────────▶ stockControl
                                                               │
                                                               ├ 재고 부족! IllegalStateException
                                                               ├ 로컬 트랜잭션 ② 롤백 (재고 그대로)
                                                               │
    stockFailure ◀────────── stock-failure-topic ◀────────────┘ StockFailedEvent
        │
        └ order.cancel() → status = CANCELLED  ★보상 트랜잭션★
```

**중요한 관찰**: 실패했을 때도 **예외를 던져서 재시도시키지 않고**, **"실패했다"는 사실을 이벤트로 만들어 보냅니다.** 이것이 SAGA의 핵심 사고방식입니다.

> 💡 **왜 실패를 예외가 아니라 이벤트로 만드나?**
> 재고 부족은 **버그가 아니라 정상적인 비즈니스 결과**입니다. 100번 재시도해도 재고는 안 생깁니다. 이런 것을 **비즈니스 실패**라 하고, DB 커넥션 끊김 같은 **기술적 실패**와 구분해야 합니다.
>
> | 실패 종류 | 예 | 처리 방법 |
> |---|---|---|
> | **비즈니스 실패** | 재고 부족, 한도 초과 | **이벤트로 알림** → 보상 트랜잭션 (재시도 무의미) |
> | **기술적 실패** | DB 커넥션 끊김, 브로커 장애 | **예외를 던져 재시도** → 실패 시 DLQ |
>
> `eda-project2`의 `StockConsumer`가 `try-catch`로 **모든 예외를 잡아 실패 이벤트로 변환**하는 것은, 이 흐름을 "비즈니스 실패" 경로로 통일한 것입니다. (다만 이렇게 하면 기술적 실패까지 흡수되어 DLQ로 못 가는 부작용이 있습니다 → Part 7에서 다룹니다.)

## 5-4. 토픽과 큐 지도

| Exchange (destination) | 발행자 | 라우팅 키 | 구독 큐 (group) | 구독자 |
|---|---|---|---|---|
| `order-created-topic` | order-service | `order.created` | `order-created-topic.stock-group` | stock-service `stockControl` |
| `stock-success-topic` | stock-service | (기본값 = 목적지명) | `stock-success-topic.order-stock-success-group` | order-service `stockSuccess` |
| `stock-failure-topic` | stock-service | (기본값 = 목적지명) | `stock-failure-topic.order-stock-failure-group` | order-service `stockFailure` |

여기에 각 큐마다 `.dlq`가 하나씩 더 생깁니다.

> 💡 **성공/실패를 왜 다른 Exchange로 나눴나?** 하나의 토픽에 라우팅 키만 다르게 해도 됩니다. 하지만 **채널을 분리하면 컨슈머 함수도 분리**되고, 각 함수는 `Consumer<StockReservedEvent>` / `Consumer<StockFailedEvent>` 처럼 **타입이 명확**해집니다. `if (성공) ... else ...` 분기가 코드에서 사라집니다. **타입 시스템이 분기를 대신 해주는** 깔끔한 설계입니다.

> 💡 **order-service는 자기가 발행한 이벤트의 결과를 다시 구독합니다.** 즉 이 서비스는 **Producer이면서 동시에 Consumer**입니다. 코레오그래피 SAGA에서 **"시작한 쪽이 최종 상태를 책임진다"** 는 원칙이 이렇게 구현됩니다.

## 5-5. 코드 정독 — order-service

### (1) `Order` 엔티티 — 상태 머신과 캡슐화

```java
@Entity @Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // ★ JPA 요구사항 + 무분별한 생성 방지
@AllArgsConstructor @Builder
public class Order {
  @Id private String orderId;
  private String productId;
  private Integer quantity;

  @Enumerated(EnumType.STRING)     // ★ ORDINAL이 아니라 STRING
  private OrderStatus status;

  public void complete() { if (this.status == OrderStatus.PENDING) this.status = OrderStatus.COMPLETED; }
  public void cancel()   { if (this.status == OrderStatus.PENDING) this.status = OrderStatus.CANCELLED; }
}
```

**설계 포인트 4가지**:

1. **`PENDING → COMPLETED / CANCELLED` 상태 머신.** SAGA에서 **"아직 결말이 안 난 중간 상태"** 를 표현하는 `PENDING`이 필수입니다. 이게 없으면 "주문은 만들어졌지만 재고 확정 전"인 순간을 표현할 수 없습니다.
2. **`@Enumerated(EnumType.STRING)`** — `ORDINAL`(기본값)은 enum을 **순서 숫자(0,1,2)** 로 저장합니다. 나중에 enum 중간에 값을 하나 끼워 넣으면 **기존 데이터가 전부 다른 의미로 뒤바뀝니다.** 실무에서는 거의 항상 `STRING`을 씁니다.
3. **`setStatus()`가 없습니다.** 대신 `complete()`, `cancel()` 이라는 **의미 있는 메서드**만 공개합니다 → 상태를 아무 값으로나 바꾸는 사고를 타입 수준에서 차단합니다 (**풍부한 도메인 모델**).
4. **`if (status == PENDING)` 가드** — Part 1-6에서 설명한 **멱등성** 장치입니다. 중복 이벤트가 와도 안전하고, 이미 취소된 주문이 나중에 COMPLETED로 뒤집히는 사고도 막습니다.

### (2) `OrderService` — 3개의 로컬 트랜잭션

```java
@Transactional
public String createOrderV1(OrderCreatedRequest request) {   // ── 로컬 트랜잭션 ①
    String orderId = UUID.randomUUID().toString();
    orderRepository.save(Order.builder()... .status(OrderStatus.PENDING).build());
    eventPublisher.publishEvent(new OrderCreatedEvent(...));   // AFTER_COMMIT 릴레이로 전달
    return orderId;
}

@Transactional
public void completeOrder(String orderId) {                  // ── 로컬 트랜잭션 ③ (성공)
    Order order = orderRepository.findById(orderId).orElseThrow(...);
    order.complete();      // ★ save() 호출이 없다!
}

@Transactional
public void cancelOrder(String orderId) {                    // ── 로컬 트랜잭션 ③' (보상)
    Order order = orderRepository.findById(orderId).orElseThrow(...);
    order.cancel();        // ★ 여기도 save() 없음
}
```

> **Q. `order.complete()`만 하고 `save()`를 안 했는데 왜 DB가 바뀌나요?**
>
> **A. JPA의 더티 체킹(Dirty Checking, 변경 감지) 때문입니다.**
>
> ```
> ① findById()로 조회 → JPA가 엔티티를 "영속성 컨텍스트(1차 캐시)"에 넣으면서
>                        조회 당시의 값을 스냅샷으로 따로 복사해 둡니다.
> ② order.complete()   → 자바 객체의 필드만 바뀝니다. (아직 DB는 그대로)
> ③ 트랜잭션 커밋 시점  → JPA가 [현재 엔티티] vs [스냅샷]을 필드별로 비교합니다.
> ④ 달라진 필드 발견   → UPDATE orders SET status='COMPLETED' WHERE order_id=?
>                        SQL을 자동 생성해서 실행(flush)합니다.
> ```
>
> **주의**: 이 마법은 **영속 상태(persistent)** 인 엔티티에만 적용됩니다. 즉 **`@Transactional` 안에서 JPA로 조회한 객체**여야 합니다. `new Order(...)`로 만든 객체나 트랜잭션 밖의 객체는 아무리 바꿔도 DB에 반영되지 않습니다.

### (3) `OrderSagaConsumer` — SAGA의 결말을 듣는 귀

```java
@Configuration @RequiredArgsConstructor
public class OrderSagaConsumer {
  private final OrderService orderService;

  @Bean
  public Consumer<StockReservedEvent> stockSuccess() {
    return event -> { orderService.completeOrder(event.orderId()); };
  }

  @Bean
  public Consumer<StockFailedEvent> stockFailure() {
    return event -> { orderService.cancelOrder(event.orderId()); };   // ★ 보상 트랜잭션
  }
}
```

**주목할 점**: 컨슈머 자체에는 비즈니스 로직이 없고 **`OrderService`에 위임**만 합니다. 이유는 **트랜잭션 경계** 때문입니다. `@Transactional`은 스프링 프록시로 동작하므로 **빈의 메서드를 외부에서 호출**해야 적용됩니다. 컨슈머 람다 안에 로직을 직접 쓰면 트랜잭션이 걸리지 않아 더티 체킹도 안 됩니다. **"컨슈머는 얇게, 서비스는 트랜잭션 단위로"** 가 정석입니다.

## 5-6. 코드 정독 — stock-service

### (1) `Stock` 엔티티 — 비즈니스 규칙을 엔티티 안에 둔다

```java
public void decrease(Integer amount) {
    if (this.quantity < amount) {
        throw new IllegalStateException(String.format("재고 부족 (요청: %d, 잔여: %d)", amount, this.quantity));
    }
    this.quantity -= amount;
}
```

재고 검증이 **서비스가 아니라 엔티티 안**에 있습니다. 이러면 **어떤 경로로 호출하든 음수 재고가 절대 만들어질 수 없습니다.** (`increase()`는 현재 미사용이지만, 결제 단계가 추가되어 **재고를 되돌리는 보상 트랜잭션**이 필요해질 때 쓸 자리입니다.)

### (2) `StockRepository` — 비관적 락 ★동시성 핵심★

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT s FROM Stock s WHERE s.productId = :productId")
Optional<Stock> findByIdWithLock(@Param("productId") String productId);
```

**왜 락이 필요한가 — 갱신 손실(Lost Update) 문제**:

```
재고 10개.  A와 B가 동시에 3개씩 주문.

 시각   스레드 A                     스레드 B
 ────   ────────────────────         ────────────────────
  t1    SELECT → quantity = 10
  t2                                 SELECT → quantity = 10   ← 똑같이 10을 읽음
  t3    10 - 3 = 7 계산
  t4                                 10 - 3 = 7 계산
  t5    UPDATE quantity = 7
  t6                                 UPDATE quantity = 7      ← A의 차감이 증발!

 결과: 6개가 팔렸는데 재고는 7개.  ❌ 1개를 초과 판매하게 됨
```

`@Lock(PESSIMISTIC_WRITE)`를 붙이면 JPA가 SQL을 이렇게 바꿉니다.

```sql
SELECT ... FROM stocks WHERE product_id = ? FOR UPDATE   -- ★ FOR UPDATE
```

`FOR UPDATE`는 **해당 행에 배타적 락**을 겁니다. B는 t2에서 **A의 트랜잭션이 커밋될 때까지 대기**하고, 깨어나서 읽으면 7을 읽습니다 → 최종 4개. ✅

**비관적 락 vs 낙관적 락**:

| | 비관적 락 (Pessimistic) ★사용 중★ | 낙관적 락 (Optimistic) |
|---|---|---|
| 철학 | "충돌은 **일어난다**. 미리 막자" | "충돌은 **드물다**. 나중에 감지하자" |
| 구현 | DB 행 락 (`SELECT ... FOR UPDATE`) | `@Version` 컬럼으로 버전 비교 |
| 충돌 시 | 대기 (blocking) | 예외 발생 → **애플리케이션이 재시도** |
| 적합 | **경합이 심한 재고/좌석/포인트** ★ | 경합이 드문 일반 수정 |
| 위험 | 대기 시간, **데드락**, 처리량 저하 | 재시도 로직을 직접 짜야 함 |

재고 차감은 **인기 상품에 요청이 몰리는(경합이 심한) 대표 사례**라 비관적 락이 적절한 선택입니다.

> 💡 락은 **트랜잭션이 끝날 때 풀립니다.** 그래서 `@Transactional decreaseStock()`의 범위를 **최대한 짧게** 유지하는 게 중요합니다. 이 코드가 조회+차감만 하고 이벤트 발행은 트랜잭션 **밖**에서 하는 이유이기도 합니다.

### (3) `StockService` — 짧고 명확한 트랜잭션

```java
@Transactional
public void decreaseStock(String productId, Integer quantity) {
    Stock stock = stockRepository.findByIdWithLock(productId)
        .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 상품 ID: " + productId));
    stock.decrease(quantity);     // 더티 체킹으로 UPDATE
}
```

이 메서드가 **정상 종료되면 커밋**, **예외가 나면 롤백**됩니다. 재고 부족 시 `IllegalStateException`이 나므로 **차감은 없던 일이 됩니다.**

### (4) `StockConsumer` — SAGA의 분기점 ★

```java
@Bean
public Consumer<OrderCreatedEvent> stockControl() {
  return event -> {
    try {
      stockService.decreaseStock(event.productId(), event.quantity());   // 트랜잭션 ②
      streamBridge.send("stockReserved-out-0", new StockReservedEvent(...));   // 성공 알림
    } catch (Exception e) {
      streamBridge.send("stockFailed-out-0", new StockFailedEvent(..., e.getMessage(), ...)); // 실패 알림
    }
  };
}
```

**여기가 이 프로젝트의 하이라이트입니다.** 한 줄씩 의미를 보면:

| 코드 | 의미 |
|---|---|
| `try { decreaseStock(...) }` | **로컬 트랜잭션 ②.** 이 메서드가 반환되는 순간 **이미 커밋이 끝났습니다** (트랜잭션 경계 = 메서드) |
| `streamBridge.send(성공)` | 커밋 **이후에** 성공 사실을 알립니다 → 유령 이벤트 방지 |
| `catch (Exception e)` | 재고 부족(`IllegalStateException`)뿐 아니라 **모든 예외**를 잡습니다. 예외를 **밖으로 던지지 않으므로 재시도/DLQ로 가지 않습니다** |
| `streamBridge.send(실패)` | **실패를 1급 시민(이벤트)으로 승격.** 이 이벤트가 주문 서비스의 보상 트랜잭션을 촉발합니다 |
| `e.getMessage()`를 실려 보냄 | 왜 실패했는지 **원인이 주문 서비스까지 전달**됩니다 → 나중에 고객 안내에 사용 가능 |

> 💡 **컨슈머가 다시 Producer가 되는 구조**에 주목하세요. 이렇게 **이벤트 → 처리 → 새 이벤트** 의 사슬이 이어지는 것이 **코레오그래피**입니다. 지휘자 없이 흐름이 완성됩니다.

## 5-7. 인프라 — `docker-compose.yml`

1번과 달리 **MySQL이 추가**되었고, 컨테이너가 뜰 때 `init.sql`이 자동 실행됩니다.

```yaml
configs:
  init_sql:
    content: |
      CREATE DATABASE IF NOT EXISTS `msa_order_db` ...;   # 주문 서비스 전용 DB
      CREATE DATABASE IF NOT EXISTS `msa_stock_db` ...;   # 재고 서비스 전용 DB
      CREATE TABLE IF NOT EXISTS `msa_stock_db`.`stocks` (
          product_id VARCHAR(255) PRIMARY KEY,
          quantity INT NOT NULL
      );
      INSERT INTO `msa_stock_db`.`stocks` VALUES ('apple', 10)
      ON DUPLICATE KEY UPDATE quantity = 10;              # 테스트용 초기 재고 10개
```

> 💡 **DB를 2개로 나눈 것이 핵심입니다.** `msa_order_db`와 `msa_stock_db`는 **논리적으로 완전히 분리된 데이터베이스**입니다. 실습 편의상 같은 MySQL 컨테이너 안에 있을 뿐, 서비스는 **자기 DB만** 봅니다. 이것을 **Database per Service** 패턴이라 하고, **MSA의 대전제**입니다.
> 이 분리가 있기 때문에 **하나의 `@Transactional`로 둘을 묶을 수 없고**, 그래서 SAGA가 필요한 것입니다. 만약 두 서비스가 같은 DB를 공유한다면 SAGA 실습 자체가 무의미해집니다.

`ddl-auto: update` 설정으로 `orders` 테이블은 order-service가 뜰 때 **JPA가 자동 생성**합니다. (`stocks`는 초기 재고 데이터가 필요해서 `init.sql`로 미리 만든 것입니다.)

> ⚠️ `ddl-auto: update`는 **학습/로컬 전용**입니다. 운영에서는 컬럼 삭제를 반영하지 않고 예측 불가한 변경을 일으키므로 `validate` + Flyway/Liquibase 같은 마이그레이션 도구를 씁니다.

## 5-8. 직접 실행하고 확인하기

```bash
cd eda-project2
docker compose up -d
./gradlew :order-service:bootRun    # 터미널 1
./gradlew :stock-service:bootRun    # 터미널 2
```

### 시나리오 A — 성공 (재고 10개 중 2개 주문)

```bash
curl -X POST http://localhost:8081/api/orders/v1 \
  -H "Content-Type: application/json" -d '{"productId":"apple","quantity":2}'
```

기대 로그:

```
[order] [Saga Started] Order Created in PENDING. Order ID: ...
[order] [Event Published to RabbitMQ] ...
[stock] [Saga Step] OrderCreatedEvent Received: ...
[stock] [Stock Deducted] Product: apple, Deducted: 2, Remaining: 8
[stock] [Event Published] StockReservedEvent -> stock-success-topic: ...
[order] [Saga Step] Received StockReservedEvent for Order ID: ...
[order] [Saga Completed] Order ID: ... -> COMPLETED
```

DB 확인:

```sql
SELECT * FROM msa_order_db.orders;   -- status = COMPLETED
SELECT * FROM msa_stock_db.stocks;   -- apple: 8
```

### 시나리오 B — 실패 & 보상 (재고보다 많이 주문)

```bash
curl -X POST http://localhost:8081/api/orders/v1 \
  -H "Content-Type: application/json" -d '{"productId":"apple","quantity":999}'
```

기대 로그:

```
[order] [Saga Started] Order Created in PENDING. Order ID: ...
[stock] [Saga Step Failed] Reason: 재고 부족 (요청: 999, 잔여: 8). Publishing StockFailedEvent
[stock] [Event Published] StockFailedEvent -> stock-failure-topic: ...
[order] [Saga Step] Received StockFailedEvent for Order ID: ... Reason: 재고 부족 ...
[order] [Saga Compensated] Order ID: ... -> CANCELLED    ★보상 트랜잭션★
```

**여기서 반드시 확인할 것**:

1. HTTP 응답은 **성공(200)** 이었습니다. 그런데 최종 주문 상태는 `CANCELLED`입니다. → **최종적 일관성(Eventual Consistency)** 을 몸으로 체감하는 지점입니다.
2. `stocks` 테이블의 재고는 **변하지 않았습니다** (로컬 트랜잭션 ②가 롤백되었으므로).
3. 즉 실제 서비스라면 **"주문 접수 → 잠시 후 취소 알림"** 같은 UX 설계가 필요합니다. EDA를 도입하면 **화면 설계까지 바뀝니다.**

### 시나리오 C — 존재하지 않는 상품

```bash
curl -X POST http://localhost:8081/api/orders/v1 \
  -H "Content-Type: application/json" -d '{"productId":"banana","quantity":1}'
```

`IllegalArgumentException`도 `catch (Exception e)`에 걸려 **똑같이 CANCELLED**로 보상됩니다.

### 시나리오 D — 시간적 분리 + SAGA

stock-service를 **끄고** 주문 → 주문은 `PENDING` 상태로 머뭅니다. stock-service를 켜면 **그제서야** 큐에 쌓인 메시지가 처리되고 `COMPLETED`로 바뀝니다. **PENDING 상태의 존재 이유**를 눈으로 확인할 수 있는 실험입니다.
---

# Part 6. 두 프로젝트 비교

## 6-1. 한눈에 보는 차이

| 항목 | `eda-project` | `eda-project2` |
|---|---|---|
| **학습 목표** | 이벤트 발행/구독 파이프라인 | **SAGA 분산 트랜잭션** |
| **데이터베이스** | 없음 (로그로 가정) | **MySQL 2개 DB** (order/stock 분리) |
| **JPA** | ❌ | ✅ (더티 체킹, 비관적 락) |
| **이벤트 종류** | 1개 (`OrderCreatedEvent`) | **3개** (+`StockReservedEvent`, `StockFailedEvent`) |
| **메시지 흐름** | **단방향** (order → stock) | **양방향 순환** (order → stock → order) |
| **Exchange 수** | 1개 | 3개 |
| **주문 상태** | 없음 | `PENDING → COMPLETED / CANCELLED` |
| **실패 처리** | 예외 → 재시도 → DLQ | **실패를 이벤트로 만들어 보상 트랜잭션** |
| **동시성 제어** | 없음 | **비관적 락** (`SELECT ... FOR UPDATE`) |
| **order-service의 역할** | Producer만 | **Producer + Consumer** |
| **stock-service의 역할** | Consumer만 | **Consumer + Producer** |
| **`function.definition`** | (없음, 발행 전용) | order: `stockSuccess;stockFailure` / stock: `stockControl` |

## 6-2. 왜 이 순서로 배우는가

```
eda-project                          eda-project2
───────────                          ────────────
"이벤트가 어떻게 흘러가는가"    ──▶   "흘러간 결과가 실패하면 어떻게 되돌리는가"

배우는 것:                            배우는 것:
· Exchange/Queue/Binding              · SAGA / 보상 트랜잭션
· 컨슈머 그룹                          · 상태 머신 (PENDING의 존재 이유)
· 재시도와 DLQ                         · 비즈니스 실패 vs 기술적 실패
· AFTER_COMMIT 릴레이 구조             · Database per Service
· 채널 네이밍 규칙                     · 더티 체킹, 비관적 락
                                      · 최종적 일관성의 실제 체감
```

**1번을 완전히 이해하지 못한 상태로 2번을 보면 코드가 미로처럼 보입니다.** 1번의 파이프라인 하나가 2번에서 **세 번 반복**될 뿐이라는 걸 알면 훨씬 단순해집니다.

## 6-3. 두 프로젝트에서 변하지 않은 구조 (= 이 아키텍처의 뼈대)

```
비즈니스 로직 (@Service, @Transactional)
        ↓ ApplicationEventPublisher (내부 이벤트)
이벤트 릴레이 (@TransactionalEventListener AFTER_COMMIT)
        ↓ StreamBridge (외부 이벤트)
Spring Cloud Stream 바인딩 (application.yml)
        ↓ 바인더
RabbitMQ
        ↓
Consumer<T> 함수 빈
```

이 5층 구조는 두 프로젝트에서 **똑같습니다.** 새 기능을 붙일 때도 이 계단을 그대로 따라가면 됩니다.

---

# Part 7. 이 구조의 한계와 다음 단계

실습 코드는 **개념 학습에 최적화**되어 있어, 운영 환경이라면 더 필요한 것들이 있습니다. 이 섹션은 "이 코드가 틀렸다"가 아니라 **"다음에 배울 것"** 의 목록입니다.

## 7-1. 이중 쓰기(Dual Write) 문제와 Outbox 패턴 ★가장 중요★

`AFTER_COMMIT`은 유령 이벤트는 막지만, **반대 방향의 사고**는 못 막습니다.

```
① DB 커밋 성공 ✅
② ...바로 이 순간 애플리케이션이 죽거나 RabbitMQ가 다운 💥
③ 이벤트는 영원히 발행되지 않음 ❌

결과: 주문은 PENDING으로 DB에 남았는데, 아무도 그 사실을 모릅니다. 영구 미아.
```

원인은 **"DB에 쓰기"와 "브로커에 쓰기"가 서로 다른 두 시스템에 대한 각각의 쓰기(=이중 쓰기)** 이고, 둘을 하나의 원자적 단위로 묶을 수 없기 때문입니다.

**해결책: Transactional Outbox 패턴**

```
[같은 DB, 같은 트랜잭션 안에서]
  ① orders 테이블에 주문 INSERT
  ② outbox 테이블에 "발행할 이벤트" INSERT   ← 같은 트랜잭션! 원자적으로 함께 커밋
  ─────────── COMMIT ───────────

[별도의 릴레이 프로세스가]
  ③ outbox 테이블을 폴링(또는 DB 변경로그 CDC를 구독)
  ④ 미발행 행을 읽어 브로커로 발행
  ⑤ 발행 성공 시 해당 행을 published로 표시
```

핵심 아이디어: **"보낼 메시지 자체를 DB 트랜잭션 안에 저장"** 해버리면, 커밋되었다는 것과 이벤트가 존재한다는 것이 **같은 사실**이 됩니다. 발행이 늦어질 수는 있어도 **유실은 없습니다.**

| | 현재 코드 (AFTER_COMMIT 릴레이) | Outbox 패턴 |
|---|---|---|
| 유령 이벤트(롤백됐는데 발행) | ✅ 막음 | ✅ 막음 |
| 이벤트 유실(커밋됐는데 미발행) | ❌ **가능** | ✅ 막음 |
| 복잡도 | 낮음 (애노테이션 1개) | 높음 (테이블 + 릴레이 프로세스) |

> 💡 현재 구조는 흔히 **"Best-effort 1-phase commit"** 이라 부릅니다. 학습/중요도 낮은 이벤트에는 충분하고, **결제·정산처럼 유실이 치명적인 곳에는 Outbox**가 필요합니다.

**추가로 볼 것**: `spring.rabbitmq.publisher-confirm-type: correlated` + `publisher-returns: true` 를 켜면 **브로커가 실제로 메시지를 받았는지 콜백으로 확인**할 수 있습니다. 현재 코드에는 없습니다.

## 7-2. 재고 차감은 멱등하지 않다

Part 1-6에서 봤듯 메시지는 **중복 전달될 수 있습니다.** 그런데:

```java
stock.decrease(quantity);   // quantity -= 2  ← 두 번 실행되면 4개가 깎임 ❌
```

`Order.complete()`는 상태 가드로 멱등하지만 **재고 차감은 방어가 없습니다.**

**해결 방법**:

```java
// 처리 이력 테이블로 중복 차단 (가장 일반적)
@Entity
class ProcessedEvent {
    @Id String eventId;        // 또는 orderId + 이벤트 타입
    Instant processedAt;
}

@Transactional
public void decreaseStock(String orderId, String productId, int qty) {
    if (processedEventRepository.existsById(orderId)) return;   // ★ 이미 처리했으면 무시
    processedEventRepository.save(new ProcessedEvent(orderId, Instant.now()));
    // ↑ PK 유니크 제약이 동시성까지 막아줍니다 (중복이면 예외 발생)
    stockRepository.findByIdWithLock(productId).orElseThrow().decrease(qty);
}
```

**핵심 원리**: 이력 저장과 재고 차감이 **같은 트랜잭션**에 있어야 합니다. 그래야 "차감은 됐는데 이력은 안 남는" 틈이 사라집니다.

> 💡 이를 위해 이벤트에 **고유 ID(`eventId`)** 를 넣는 것이 정석입니다. 현재 이벤트에는 `orderId`가 있어 이를 대신 쓸 수 있습니다.

## 7-3. `catch (Exception e)`가 너무 넓다

```java
catch (Exception e) {   // ← 재고 부족도, DB 커넥션 끊김도, NPE도 모두 여기로
    streamBridge.send("stockFailed-out-0", ...);
}
```

Part 5-3에서 구분한 두 종류의 실패가 **하나로 뭉개집니다.**

| 실제로 일어난 일 | 지금의 동작 | 바람직한 동작 |
|---|---|---|
| 재고 부족 | 실패 이벤트 발행 → 주문 취소 ✅ | 동일 ✅ |
| **DB가 잠깐 끊김** | 실패 이벤트 발행 → **주문이 부당하게 취소됨** ❌ | 예외를 던져 **재시도** → 안 되면 DLQ |

또한 예외를 밖으로 안 던지므로 `max-attempts`와 DLQ 설정이 **사실상 작동하지 않습니다.**

**개선 방향**:

```java
try {
    stockService.decreaseStock(...);
    streamBridge.send("stockReserved-out-0", ...);
} catch (IllegalStateException | IllegalArgumentException e) {   // 비즈니스 실패만
    streamBridge.send("stockFailed-out-0", ...);
}
// 그 외 기술적 예외는 잡지 않고 밖으로 → 재시도 → DLQ
```

> 💡 이를 위해 도메인 예외를 따로 정의(`OutOfStockException` 등)하고, **비즈니스 예외 계층과 기술 예외 계층을 분리**하는 것이 실무의 정석입니다.

## 7-4. SAGA에 타임아웃이 없다

stock-service가 영영 안 살아나면 주문은 **`PENDING`으로 영원히 고착**됩니다. 아무도 이를 감지하지 않습니다.

**해결 방향**:

- 주문에 `createdAt`을 두고, **스케줄러가 주기적으로 "N분 이상 PENDING인 주문"을 찾아** 강제 취소하거나 운영자에게 알림
- 또는 **SAGA 상태 테이블**을 따로 두고 각 단계의 진행/타임아웃을 관리 (오케스트레이션 방식으로 진화)

## 7-5. 순서 보장이 없다

같은 주문에 대해 `OrderCreated`와 (미래에 추가될) `OrderCancelled` 이벤트가 **순서가 뒤바뀌어 도착**할 수 있습니다. 컨슈머 인스턴스가 여러 대면 특히 그렇습니다.

**해결 방향**: 파티셔닝(같은 `orderId`는 항상 같은 인스턴스로), 또는 이벤트에 **버전/시퀀스 번호**를 넣고 오래된 이벤트는 무시.

## 7-6. 관측성(Observability)이 없다

로그가 서비스마다 흩어져 있어 **"이 주문이 어디서 막혔나"** 를 추적하기 어렵습니다.

**해결 방향**:

- **분산 추적**: Micrometer Tracing + Zipkin/Tempo — `traceId`가 HTTP → 메시지 헤더 → 다음 서비스까지 자동 전파됩니다.
- **상관관계 ID**: 이벤트에 `correlationId`를 넣고 모든 로그에 찍기 (MDC 활용).
- **DLQ 모니터링**: DLQ에 메시지가 쌓이면 알람 (현재는 아무도 안 봄).

## 7-7. 코레오그래피의 한계 — 언제 오케스트레이션으로 가야 하나

지금은 서비스가 2개라 흐름이 명확합니다. 그런데 **주문 → 재고 → 결제 → 배송 → 포인트** 로 늘어나면:

- 전체 흐름이 **어느 코드에도 적혀 있지 않습니다.** 5개 서비스 코드를 다 열어봐야 흐름을 알 수 있습니다.
- 3단계에서 실패하면 **1, 2단계를 역순으로 보상**해야 하는데, 그 조율을 아무도 안 합니다.
- 서비스 간 이벤트가 **순환 참조**를 만들기 쉽습니다.

**보통 참여 서비스가 4개를 넘으면 오케스트레이션**(중앙 SAGA 오케스트레이터, 예: Camunda, Temporal, 또는 직접 구현한 상태 머신)으로 전환합니다.

## 7-8. 그 밖의 항목

| 항목 | 현재 | 개선 |
|---|---|---|
| DB 비밀번호 | YAML에 평문 `1234` | 환경변수 / Vault / K8s Secret |
| `ddl-auto: update` | 스키마 자동 변경 | `validate` + Flyway/Liquibase |
| 공용 DTO 모듈 | 모든 서비스가 함께 결합 | 스키마 레지스트리, 하위 호환 규칙 |
| 테스트 | 없음 | Testcontainers(RabbitMQ/MySQL 실제 기동) + `spring-cloud-stream-test-binder` |
| 서비스 디스커버리/게이트웨이 | 없음 (포트 직접 지정) | Eureka/K8s Service + Spring Cloud Gateway |
| 이벤트 스키마 버전 | 없음 | 이벤트에 `version` 필드, 하위 호환 강제 |

---

# Part 8. 용어 사전

| 용어 | 뜻 |
|---|---|
| **EDA** | Event-Driven Architecture. 서비스들이 직접 호출 대신 **이벤트를 주고받아** 협력하는 아키텍처 |
| **이벤트 (Event)** | **이미 일어난 사실**을 담은 불변 메시지. 이름은 과거형 (`OrderCreated`) |
| **명령 (Command)** | 특정 대상에게 **무언가를 하라고 시키는** 메시지. 이름은 명령형 (`CreateOrder`) |
| **Producer / Publisher** | 메시지를 만들어 브로커에 보내는 쪽 |
| **Consumer / Subscriber** | 브로커에서 메시지를 꺼내 처리하는 쪽 |
| **Broker (브로커)** | 메시지를 중계·보관하는 미들웨어. 여기서는 RabbitMQ |
| **AMQP** | RabbitMQ가 구현한 메시징 표준 프로토콜 |
| **Exchange (익스체인지)** | 메시지를 받아 **어느 큐로 보낼지 결정**하는 라우터. 저장하지 않음 |
| **Queue (큐)** | 메시지가 **쌓여 대기**하는 저장소 |
| **Binding (바인딩)** | Exchange와 Queue를 잇는 **규칙**. 조건으로 라우팅 키 패턴을 씀 |
| **Routing Key** | 메시지에 붙는 **주소 라벨**. 예: `order.created` |
| **Topic Exchange** | 라우팅 키를 **와일드카드(`*`, `#`)로 패턴 매칭**하는 Exchange 타입 |
| **Consumer Group** | 같은 그룹의 인스턴스들이 **하나의 큐를 나눠 먹음** → 중복 처리 방지 |
| **경쟁 소비자 패턴** | 여러 인스턴스가 같은 큐에서 메시지를 **가져가려고 경쟁**하는 구조 |
| **ACK / NACK** | 메시지 처리 성공/실패를 브로커에 알리는 응답. ACK 후에야 큐에서 삭제됨 |
| **DLQ** | Dead Letter Queue. 재시도를 다 소진한 **실패 메시지를 격리 보관**하는 큐 |
| **Poison Message** | 무한히 실패하며 큐를 막는 **독약 메시지** |
| **At-least-once** | **유실은 없지만 중복은 가능**한 전달 보장. RabbitMQ 기본 |
| **멱등성 (Idempotency)** | 같은 작업을 여러 번 해도 **결과가 한 번 한 것과 같은** 성질 |
| **Spring Cloud Stream** | 브로커를 추상화해 **함수 빈만으로** 메시징을 구현하게 해주는 프레임워크 |
| **Binder (바인더)** | SCSt의 **브로커 어댑터**. 의존성 교체만으로 RabbitMQ ↔ Kafka 전환 |
| **Binding (SCSt)** | 함수의 입출력 채널과 브로커 목적지를 잇는 **설정** |
| **destination** | SCSt의 논리적 목적지. RabbitMQ에서는 **Exchange 이름** |
| **StreamBridge** | 코드 원하는 지점에서 **명령형으로 메시지를 발행**하는 도구 |
| **`ApplicationEventPublisher`** | 스프링 컨테이너 **내부**에서만 도는 인메모리 이벤트 버스. 네트워크 안 탐 |
| **`@TransactionalEventListener`** | 내부 이벤트를 **트랜잭션 특정 시점(커밋 후 등)** 에 처리하는 리스너 |
| **유령 이벤트 (Phantom Event)** | **롤백되어 없던 일이 된 작업**에 대해 발행되어 버린 이벤트 |
| **이중 쓰기 (Dual Write)** | DB와 브로커, **두 시스템에 각각 써야 해서** 원자성이 깨지는 문제 |
| **Outbox 패턴** | 발행할 이벤트를 **DB 트랜잭션 안 테이블에 저장**한 뒤 별도 프로세스가 발행 |
| **분산 트랜잭션** | 여러 DB/서비스에 걸친 하나의 논리적 작업 |
| **2PC** | Two-Phase Commit. 조정자가 준비→커밋 2단계로 묶는 방식. MSA엔 부적합 |
| **SAGA** | 분산 트랜잭션을 **로컬 트랜잭션 여러 개 + 보상 트랜잭션**으로 푸는 패턴 |
| **보상 트랜잭션** | 이전 단계를 **되돌리는 새로운 작업**. 롤백과 달리 흔적이 남음 |
| **코레오그래피** | 지휘자 없이 **각 서비스가 이벤트를 듣고 스스로** 다음을 결정 |
| **오케스트레이션** | 중앙 조정자가 각 서비스에 **명령을 내려** 흐름을 통제 |
| **최종적 일관성** | 잠깐은 서로 안 맞지만 **결국엔 일치**하게 되는 일관성 모델 |
| **Database per Service** | 서비스마다 **자기 DB를 독점**하는 MSA 원칙 |
| **더티 체킹** | JPA가 커밋 시점에 **스냅샷과 비교해 UPDATE를 자동 생성**하는 기능 |
| **영속성 컨텍스트** | JPA가 엔티티를 관리하는 **1차 캐시 공간**. 트랜잭션 범위로 동작 |
| **비관적 락** | `SELECT ... FOR UPDATE`로 **미리 행을 잠그는** 동시성 제어 |
| **낙관적 락** | `@Version` 컬럼으로 **충돌을 나중에 감지**하는 동시성 제어 |
| **갱신 손실 (Lost Update)** | 두 트랜잭션이 같은 값을 읽고 각자 수정해 **한쪽 수정이 사라지는** 현상 |
| **가상 스레드** | Java 21의 경량 스레드. 블로킹 시 OS 스레드에서 **언마운트**되어 처리량 향상 |
| **BOM** | Bill of Materials. **라이브러리 버전을 한 곳에서 관리**하는 의존성 목록 |

---

# Part 9. 실행·검증 치트시트

## 9-1. 사전 준비

| 필요한 것 | 확인 방법 |
|---|---|
| JDK 21 | `java -version` |
| Docker Desktop | `docker ps` |
| (Gradle은 wrapper 포함이라 설치 불필요) | `./gradlew -v` |

## 9-2. 실행 순서

```bash
# ── eda-project (1번) ──────────────────────────
cd eda-project
docker compose up -d                 # RabbitMQ 기동
./gradlew :order-service:bootRun     # 터미널 1
./gradlew :stock-service:bootRun     # 터미널 2

# ── eda-project2 (2번) ─────────────────────────
cd eda-project2
docker compose up -d                 # RabbitMQ + MySQL 기동
docker compose ps                    # 두 컨테이너가 healthy 인지 확인 (특히 MySQL)
./gradlew :order-service:bootRun     # 터미널 1
./gradlew :stock-service:bootRun     # 터미널 2
```

> ⚠️ 두 프로젝트는 **RabbitMQ 포트(5672)와 서비스 포트(8081/8082)가 겹칩니다.** 한 번에 하나씩만 실행하세요. 전환할 때는 이전 프로젝트에서 `docker compose down` 을 먼저 실행합니다.

## 9-3. 주요 접속 정보

| 대상 | 주소 / 접속법 |
|---|---|
| 주문 서비스 | http://localhost:8081 |
| 재고 서비스 | http://localhost:8082 |
| RabbitMQ 관리 콘솔 | http://localhost:15672 (guest / guest) |
| MySQL (2번만) | `localhost:3306`, root / `1234` |

## 9-4. 요청 예시

```bash
# 정상 주문
curl -X POST http://localhost:8081/api/orders/v1 \
  -H "Content-Type: application/json" \
  -d '{"productId":"apple","quantity":2}'

# 재고 부족 (2번 프로젝트에서 보상 트랜잭션 확인)
curl -X POST http://localhost:8081/api/orders/v1 \
  -H "Content-Type: application/json" \
  -d '{"productId":"apple","quantity":999}'
```

## 9-5. DB 확인 (2번 프로젝트)

```bash
docker exec -it mysql-container mysql -uroot -p1234
```

```sql
SELECT order_id, product_id, quantity, status FROM msa_order_db.orders;
SELECT * FROM msa_stock_db.stocks;

-- 재고 초기화 (테스트 반복용)
UPDATE msa_stock_db.stocks SET quantity = 10 WHERE product_id = 'apple';
```

## 9-6. RabbitMQ 관리 콘솔에서 볼 것

| 탭 | 확인 포인트 |
|---|---|
| **Exchanges** | `order-created-topic`, `stock-success-topic`, `stock-failure-topic` 이 **자동 생성**됨 |
| **Exchanges → 이름 클릭 → Bindings** | 어떤 큐가 **어떤 라우팅 키로** 연결됐는지 |
| **Queues** | `...stock-group` 과 `...stock-group.dlq` 쌍 확인. `Ready` 열이 **대기 중인 메시지 수** |
| **Queues → 큐 클릭 → Get messages** | 실제 JSON 페이로드와 헤더(`contentType`, 실패 시 `x-exception-*`)를 눈으로 확인 |

## 9-7. 문제가 생겼을 때 체크리스트

| 증상 | 확인할 것 |
|---|---|
| **메시지가 컨슈머에 안 옴** | ① 함수 빈 이름 == `<이름>-in-0` 인가? ② `spring.cloud.function.definition`에 함수 이름을 넣었는가? ③ 발행/수신의 `destination`이 같은가? ④ **라우팅 키가 `binding-routing-key` 패턴에 매칭되는가?** |
| **주문은 되는데 이벤트가 안 나감** | `@TransactionalEventListener`가 커밋을 기다리는데 **트랜잭션이 없거나 롤백**되지 않았는지. (1번 프로젝트의 Part 4-2 (3) 주의사항 참고) |
| **역직렬화 실패** | `common-module`의 이벤트 클래스가 **양쪽에서 동일한지**, 필드 이름이 바뀌지 않았는지 |
| **큐가 안 생김** | 애플리케이션이 실제로 떴는지, `destination`/`group` 오타 확인. **컨슈머 쪽이 떠야 큐가 만들어집니다** |
| **재시작 후 설정이 안 먹음** | RabbitMQ에 이미 **다른 속성으로 만들어진 큐/Exchange**가 있으면 충돌합니다. `docker compose down -v` 로 초기화 |
| **MySQL 연결 실패 (2번)** | `docker compose ps` 로 healthy 확인. 초기 기동에 10~20초 걸립니다 |
| **재고가 계속 부족하다고 나옴** | 이전 테스트로 이미 차감된 상태. 9-5의 초기화 SQL 실행 |

## 9-8. 학습 확인 문제 (스스로 답해보기)

1. `publishEvent()`와 `streamBridge.send()`는 각각 어디로 메시지를 보내는가? 왜 두 단계로 나눴는가?
2. `group: stock-group` 을 지우면 재고 서비스를 3대 띄웠을 때 무슨 일이 생기는가?
3. `routing-key-expression`을 지우면 1번 프로젝트에서 메시지는 어디로 가는가? 왜 에러가 안 나는가?
4. `completeOrder()`에 `save()` 호출이 없는데 DB가 바뀌는 이유는?
5. 2번 프로젝트에서 `@Lock(PESSIMISTIC_WRITE)`를 지우고 동시에 100건을 주문하면 무엇이 깨지는가?
6. `StockConsumer`의 `catch (Exception e)`를 지우면 재고 부족 시 어떤 일이 벌어지는가? (힌트: 재시도와 DLQ)
7. HTTP 응답은 200인데 최종 주문 상태가 `CANCELLED`일 수 있다. 이걸 뭐라고 부르며, 실제 서비스라면 UX를 어떻게 설계해야 하는가?
8. 커밋은 됐는데 RabbitMQ가 그 순간 다운되면 이벤트는? 이를 막는 패턴의 이름은?

---

## 마무리 — 이 실습에서 가져갈 3가지

1. **결합을 끊는 방법은 "호출하지 않는 것"이다.** 주문 서비스는 재고 서비스의 존재를 모릅니다. Exchange가 그 사이를 대신 이어줍니다.
2. **트랜잭션 경계와 이벤트 발행 시점은 반드시 함께 설계해야 한다.** `AFTER_COMMIT` 릴레이 → (더 나아가) Outbox 패턴으로 이어지는 흐름이 EDA 설계의 척추입니다.
3. **분산 환경에서 롤백은 없다. 보상만 있다.** SAGA는 "되돌리기"가 아니라 **"되돌리는 새 작업을 실행하기"** 이며, 그래서 `PENDING` 같은 중간 상태와 최종적 일관성이 필수가 됩니다.
