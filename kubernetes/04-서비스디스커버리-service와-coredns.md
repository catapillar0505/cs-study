> [← kubernetes 목차](README.md)

# 서비스 디스커버리 — Service + CoreDNS

## 관통하는 원리

> **IP는 변한다. 이름은 안 변한다. 그래서 이름으로 부르고, 이름→IP 변환은 시스템이 알아서 한다.**

이것이 **간접 계층(indirection layer)**. 노드 이름 식별도, 서비스 디스커버리도 이 원리의 서로 다른 적용.

## 왜 IP로 직접 부르면 안 되나

```
Deployment 재배포  → 기존 Pod 삭제, 새 Pod 생성 → IP 완전히 바뀜
Pod 죽어서 재시작  → 새 IP
스케일 아웃 3→5    → IP 2개 추가
노드 장애로 이동   → 다른 노드에서 뜨며 IP 바뀜
```

**Pod IP는 태어날 때 배정받고 죽으면 사라지는 극단적으로 수명이 짧은 값.**

```java
// 재배포 한 번에 전부 터진다
restTemplate.getForObject("http://10.244.1.5:8080/pay", ...);
```

## 부품 ① Service — 안 변하는 가짜 IP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service     # ← 영원히 안 변함
spec:
  selector:
    app: payment            # ← 이 라벨 달린 Pod들을 찾아라
  ports:
    - port: 8080
```

**ClusterIP**를 하나 받는다 (예: `10.96.0.42`).

- Service가 삭제될 때까지 **절대 안 바뀜**
- 근데 **실체가 없음.** 저 IP를 가진 랜카드도 프로세스도 없다.

**어떻게 동작하나 — iptables로 낚아챈다:**

```
Pod가 10.96.0.42:8080 으로 패킷 전송
        ↓
kube-proxy가 심어둔 iptables 규칙이 가로챔
        ↓
DNAT: 목적지를 실제 Pod IP로 바꿔치기 (여러 개면 랜덤 = 로드밸런싱)
        ↓
10.244.2.7:8080 도착
```

> **★ 커널 설정과 완전히 연결되는 지점**
> `bridge-nf-call-iptables = 1` 을 켠 이유가 정확히 이것.
> Pod가 브리지로 패킷을 보낼 때 iptables를 안 거치면 **이 낚아채기가 발동 자체를 안 한다.**
> Service IP로 보낸 패킷이 허공으로 사라진다.

Pod가 죽고 새로 뜨면? kube-proxy가 **iptables 규칙만 갱신**. Service IP는 그대로. 호출하는 쪽은 아무것도 몰라도 된다.

## 부품 ② CoreDNS — 이름을 IP로

```java
// 이렇게 쓸 수 있게 됨
restTemplate.getForObject("http://payment-service:8080/pay", ...);
```

**Service를 만들면 CoreDNS에 DNS 레코드가 자동 등록**된다. 별도 설정 불필요.

```
payment-service.default.svc.cluster.local
└─ Service명 ─┘ └네임스페이스┘ └── 고정 ──┘
```

같은 네임스페이스면 `payment-service` 만 써도 된다. Pod의 `/etc/resolv.conf`에 검색 도메인이 자동 설정되어 있기 때문.

```bash
# Pod 안에서
cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
```

## ★ 최종 흐름 — 간접 계층 2겹

```
"payment-service"
  → [CoreDNS]  → 10.96.0.42   (Service ClusterIP, 안 변함)
  → [iptables] → 10.244.2.7   (실제 Pod IP, 계속 변함)
```

**이름 → 안 변하는 가짜 IP → 계속 변하는 진짜 IP**

## 노드 이름 vs 서비스 디스커버리 — 뭐가 다른가

| | 노드 이름 | 서비스 디스커버리 |
|---|---|---|
| 목적 | **객체 식별** (누구인지) | **통신 대상 찾기** (어디로 보낼지) |
| 이름의 역할 | DB의 PK 같은 것 | DNS 조회 키 |
| 변환 메커니즘 | 없음 (Node 객체의 속성) | CoreDNS + iptables |
| 로드밸런싱 | 없음 | 있음 |

> **Java 비유**
> - 노드 이름 = `@Id private String name;` — 엔티티 식별자
> - 서비스 디스커버리 = `@Autowired PaymentClient client;` — 구현체를 몰라도 이름으로 주입받아 쓰는 것
>
> DI 컨테이너와 구조가 같다: **"구체적 인스턴스를 직접 new 하지 말고, 이름으로 요청하면 컨테이너가 찾아준다."**

## Spring Cloud 대응표

| Spring Cloud | Kubernetes | 하는 일 |
|---|---|---|
| Eureka Server | **CoreDNS + etcd** | 서비스 목록 저장소 |
| Eureka Client 등록 | **자동** (Pod 뜨면 Endpoints 등록) | 나 여기 있다고 알림 |
| `@LoadBalanced RestTemplate` | **iptables (kube-proxy)** | 인스턴스 중 하나 선택 |
| Ribbon | **kube-proxy** | 로드밸런싱 |
| `http://payment-service/pay` | `http://payment-service/pay` | **문법이 똑같음!** |

**핵심 차이**: Spring Cloud는 **애플리케이션 레벨**(라이브러리)에서, k8s는 **인프라 레벨**(커널)에서 처리.

→ k8s를 쓰면 Eureka가 필요 없어진다. 언어도 안 가린다. Java든 Python이든 `http://payment-service` 로 부르면 끝. **라이브러리 의존성 0.**

## 예외 — Headless Service

"로드밸런싱 말고 **개별 Pod IP를 전부 알고 싶다**" 는 경우 (DB 클러스터, Kafka 등).

```yaml
spec:
  clusterIP: None    # ← headless
```

ClusterIP를 안 만들고 DNS 조회 시 **Pod IP 목록을 통째로 반환**:

```
nslookup my-db
→ 10.244.1.5
  10.244.2.7
  10.244.3.9
```

StatefulSet과 함께 쓰면 Pod마다 고유 DNS 이름이 생긴다:
```
my-db-0.my-db.default.svc.cluster.local
my-db-1.my-db.default.svc.cluster.local
```

**여기선 이름이 곧 개별 신원** — 노드 이름과 성격이 비슷해지는 케이스.

---

[← kubectl context와 kubeconfig](03-kubectl-context와-kubeconfig.md) | [워크로드 오브젝트 — Deployment / ReplicaSet / Service →](05-워크로드-deployment-replicaset-service.md)
