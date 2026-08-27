> [← kubernetes 목차](README.md)

# 워크로드 오브젝트 — Deployment / ReplicaSet / Service

## 왜 Pod만으로는 안 되나 — 사고 3종

Pod를 직접 하나 띄우면 이런 일이 생긴다.

| 사고 | 내용 | 해결책 |
|---|---|---|
| ① **부활 불가** | Pod가 죽으면 아무도 안 살려줌 | ReplicaSet |
| ② **수동 확장** | 3개로 늘리려면 손으로 3번 | ReplicaSet |
| ③ **배포 중단** | 1.0 지우고 2.0 띄우는 사이 서비스 끊김 | Deployment |
| ④ **IP 변경** | 배포하면 Pod IP가 전부 바뀜 | Service |

## 카페 비유

| k8s | 카페 | 하는 일 |
|---|---|---|
| **Pod** | 알바생 한 명 | 실제로 커피를 만드는 사람 |
| **ReplicaSet** | 인원 관리 규칙 | **"항상 3명"** 을 지킴 |
| **Deployment** | 사장님 | 알바를 **천천히 교체**, 문제 생기면 되돌림 |
| **Service** | 가게 대표 전화번호 | 손님은 이 번호만 알면 됨 |

---

## 8-1. ReplicaSet — 개수를 지키는 감시자

하는 일은 **딱 하나: 개수 세기.**

```yaml
replicas: 3
```

```
[감시] 지금 몇 개? → 3개. 아무것도 안 함.
[감시] 지금 몇 개? → 2개! (하나 죽음) → 즉시 1개 생성
[감시] 지금 몇 개? → 4개! (실수로 추가) → 즉시 1개 삭제
```

### ★ 선언형(Declarative) — k8s 전체를 관통하는 사고방식

| | 명령형 (Imperative) | 선언형 (Declarative) |
|---|---|---|
| 말하는 법 | "Pod 하나 만들어" | "Pod가 3개인 **상태를 유지**해" |
| 누가 관리 | 내가 계속 봐야 함 | **시스템이 알아서** |
| 비유 | "에어컨 켜" | "온도 24도 유지해" |

**원하는 상태(desired state)만 적으면 k8s가 현재 상태를 거기에 맞춘다.** 이 루프를 계속 도는 게 **컨트롤러**.

> **실무 주의**: ReplicaSet은 직접 만들 일이 거의 없다. **Deployment가 대신 만들어준다.**

---

## 8-2. Deployment — 버전을 갈아끼우는 사장님

ReplicaSet은 개수만 셀 줄 알지 **버전 교체는 못 한다.** 그래서 그 위에 Deployment가 있다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:            # ← 이 아래가 "Pod 설계도"
    metadata:
      labels:
        app: my-app    # ★ 이 라벨이 Service와 연결되는 지점
    spec:
      containers:
      - name: my-app
        image: my-app:1.0
```

### ★ 소유 관계 (가장 헷갈리는 부분)

```
Deployment (my-app)
   │ 만들고 관리함
   ▼
ReplicaSet (my-app-7d9f8b)      ← 버전 1.0 담당
   │ 만들고 관리함
   ▼
Pod, Pod, Pod                    ← 실제로 도는 놈들
```

```bash
kubectl get deploy,rs,pod
# deployment.apps/my-app            3/3
# replicaset.apps/my-app-7d9f8b     3      ← Deployment명 + 해시
# pod/my-app-7d9f8b-abc12           1/1    ← ReplicaSet명 + 해시
```

**이름만 봐도 족보가 보인다.**

### 롤링 업데이트 — Deployment의 존재 이유

```bash
kubectl set image deployment/my-app my-app=my-app:2.0
```

Deployment는 **ReplicaSet을 새로 하나 더 만들어** 이렇게 움직인다.

```
시작:  RS(1.0) ●●●        RS(2.0)
       ↓ 새 거 하나 띄우고
       RS(1.0) ●●●        RS(2.0) ○
       ↓ 준비되면 옛날 거 하나 죽이고
       RS(1.0) ●●         RS(2.0) ○
       ↓ 반복
끝:    RS(1.0)            RS(2.0) ○○○
```

**한 번도 0개가 된 적이 없다** → 무중단 배포.

> 알바 3명을 교체할 때 다 자르고 새로 뽑으면 그날 가게 문을 닫아야 한다. **한 명씩 인수인계하며 바꾸는 것.**

### 롤백 — 옛날 ReplicaSet을 안 지우는 이유

```bash
kubectl rollout undo deployment/my-app
```

**옛 ReplicaSet을 지우지 않고 `replicas=0`으로만 만들어두기 때문에** 가능하다.

```bash
kubectl get rs
# my-app-7d9f8b   0    ← 1.0, 껍데기로 남아있음
# my-app-5c8e2a   3    ← 2.0, 현재 활성
```

> **Git 비유**: 옛 커밋을 안 지우니까 `revert`가 되는 것과 같다. ReplicaSet이 **버전별 스냅샷** 역할.

기본 히스토리 보관 개수는 10개.

```bash
kubectl rollout history deployment/my-app   # 배포 이력
kubectl rollout status deployment/my-app    # 진행 상황 실시간
```

---

## 8-3. Service — 안 변하는 대표 전화번호

### 문제: 배포하면 Pod IP가 전부 바뀐다

```
배포 전: 10.244.1.5, 10.244.1.6, 10.244.1.7
배포 후: 10.244.2.11, 10.244.2.12, 10.244.2.13   ← 전부 다름
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app        # ← Deployment template의 labels와 일치해야 함
  ports:
    - port: 80         # Service로 들어오는 포트
      targetPort: 8080 # Pod로 전달할 포트
```

### ★ 라벨(label) — k8s에서 가장 중요한 연결 방식

Service는 Pod의 이름도 IP도 모른다. 그냥 **"`app: my-app` 라벨 붙은 애들 다 나와"** 하고 외친다.

```
Service의 selector:  app: my-app
                        ↓ 매칭
Pod의 labels:        app: my-app  ✅ 포함
                     app: other   ❌ 제외
```

> **비유**: "노란 조끼 입은 사람 다 모여!" 이름을 안 불러도 조끼만 보고 모인다. 새 알바가 노란 조끼를 입으면 **자동으로 그룹에 포함**된다.

그래서 Pod가 뜨든 죽든 **Service 설정은 손댈 필요가 없다.**

```bash
kubectl get endpoints my-app-service
# ENDPOINTS
# 10.244.2.11:8080,10.244.2.12:8080,10.244.2.13:8080
```
> 여기가 **비어 있으면 라벨 오타**다. 가장 흔한 실수.

### Service 종류

| 종류 | 접근 범위 | 언제 |
|---|---|---|
| **ClusterIP** (기본) | 클러스터 **안**에서만 | 백엔드끼리 통신 |
| **NodePort** | 노드IP:포트로 외부에서 | 테스트용 |
| **LoadBalancer** | 클라우드 LB 경유 | 실서비스 (AWS면 ELB 자동 생성) |

---

## 8-4. 전체 그림

```
        [외부 사용자]
              │
              ▼
     ┌─────────────────┐
     │    Service      │  ← 안 변하는 주소 (10.96.0.42)
     │  app: my-app    │     라벨로 Pod를 찾음
     └────────┬────────┘
              │ 라벨 매칭 (소유 관계 아님!)
    ┌─────────┼─────────┐
    ▼         ▼         ▼
  [Pod]     [Pod]     [Pod]     ← 실제 앱 (IP 계속 변함)
    ▲         ▲         ▲
    └─────────┼─────────┘
              │ 소유 (개수 유지)
      ┌───────────────┐
      │  ReplicaSet   │  ← "3개 지켜"
      └───────┬───────┘
              │ 소유 (버전 관리)
      ┌───────────────┐
      │  Deployment   │  ← "2.0으로 무중단 교체"
      └───────────────┘
```

> **⚠️ 핵심**: Service는 Deployment와 **직접 연결되어 있지 않다.** 오직 **라벨로만** Pod를 찾는다. Deployment를 지워도 Service는 남아 있고, 라벨만 맞으면 아무 Pod나 붙는다.

## 8-5. Spring 치환표

| k8s | Spring | 공통점 |
|---|---|---|
| Deployment | 배포 설정 | 어떻게 만들지 선언 |
| ReplicaSet | **커넥션 풀 (min/max)** | **개수를 유지**하는 관리자 |
| Pod | 실제 인스턴스 | 일하는 놈 |
| Service | `@Autowired` / DI 컨테이너 | **구현체를 몰라도 이름으로** 접근 |
| 라벨 셀렉터 | `@Qualifier` | 이름표로 대상 특정 |

> HikariCP가 죽은 커넥션을 알아서 채워 최소 개수를 유지하는 것 = ReplicaSet의 동작 원리와 동일.

---

## 8-6. ★ "관리한다"는 게 구체적으로 무엇인가

추상적으로 들리는 이 말의 실체는 **딱 두 동작 — 세기와 만들기/지우기**다.

```java
while (true) {
    // ① 세기 — selector 사용
    List<Pod> 내Pod들 = allPods.stream()
        .filter(p -> p.labels.containsAll(spec.selector.matchLabels))
        .toList();

    int 현재 = 내Pod들.size();
    int 목표 = spec.replicas;

    // ② 맞추기 — template 사용
    if (현재 < 목표)  createPod(spec.template);
    else if (현재 > 목표) deletePod(내Pod들.get(0));

    sleep();
}
```

**이 루프가 전부다.**

| 필드 | 루프 어디서 쓰이나 |
|---|---|
| `selector` | **①번 — 셀 때** (누가 내 Pod인지 판단) |
| `template` | **②번 — 만들 때** (뭘 만들지) |
| `replicas` | 목표 숫자 |

> **selector는 "세는 기준", template은 "만드는 설계도".** 쓰이는 시점이 완전히 다르다.

### 시나리오별 동작

| 사건 | 루프의 반응 |
|---|---|
| 최초 생성 | 0개 → 3개 부족 → template으로 3개 생성 |
| Pod 삭제 | 2개 → 1개 부족 → 1개 생성 (수 초 내) |
| **노드 장애** | 정상 1개 → 2개 부족 → 다른 노드에 2개 생성 (**자가 치유**) |
| `scale --replicas=5` | 3개 → 2개 생성 |

**자가 치유는 별도 기능이 아니라 "세고 맞추기"의 부산물이다.**

### 라벨을 떼면 — 이게 이해의 열쇠

```bash
kubectl label pod nginx-deploy-7d9f8b-abc12 app-
```
```
루프: app=nginx-deploy인 Pod 세기 → 2개! (뗀 Pod는 안 세어짐)
     1개 부족 → 새로 1개 생성
결과: Pod가 총 4개 (고아 1 + 정상 3)
```

> **결정적 사실: Deployment는 자기가 만든 Pod를 기억하지 않는다.** 매번 라벨로 다시 센다.
> "누가 내 Pod인가?" = **"지금 이 라벨을 달고 있는가"**. 출생 기록이 아니라 **현재 상태**로 판단한다.

실무에서 **디버깅 기법**으로 쓴다. 문제 있는 Pod의 라벨만 떼어 트래픽에서 빼고 살려둔 채 분석한다. 이때 Deployment를 지워도 고아 Pod는 살아남는다.

### template은 언제 쓰이나 — Pod를 새로 만들 때만

```bash
kubectl set image deployment/nginx-deploy nginx-container=nginx:1.25
```
**기존 Pod가 변신하는 게 아니다.** 새 template으로 새 Pod를 만들고 옛 Pod를 지운다.

> **Pod는 사실상 불변(immutable)이다.** 이미지도 환경변수도 수정되지 않는다. 바꾸려면 죽이고 새로 만드는 것뿐 — **불변 인프라(immutable infrastructure)** 원칙.
>
> **Java 비유**: template은 **생성자 인자**다. 객체 생성 시에만 쓰이고 기존 객체를 바꾸지 않는다. `String`이 불변이라 `replace()`가 새 객체를 반환하는 것과 같다.

### Deployment는 Pod를 직접 안 건드린다

```
① Deployment: "template이 바뀌었네. 새 RS 만들자"
② RS(new) 생성, replicas=0
③ Deployment: RS(new)=1, RS(old)=2     ← 숫자만 조절
④ 각 RS가 자기 루프로 Pod 조절
⑤⑥ 반복 → RS(new)=3, RS(old)=0
```

**Deployment는 ReplicaSet의 숫자만 조절하고, 실제 Pod 생성/삭제는 각 RS가 자기 루프로 한다.**

---

## 8-7. pause vs Service — 둘 다 "IP 안정성"인데 뭐가 다른가

> **pause = Pod가 살아있는 동안 IP 유지**
> **Service = Pod가 죽어도 접근 주소 유지**

| 사건 | Pod IP는? | 누가 막아주나 |
|---|---|---|
| 앱 컨테이너 크래시 → 재시작 | 안 바뀜 | **pause** |
| Pod 삭제 → 재생성 | **바뀜** | pause 무력 |
| 롤링 업데이트 | **바뀜** | pause 무력 |
| 스케일 아웃 | 새 IP 추가 | pause 무력 |
| 노드 장애로 이동 | **바뀜** | pause 무력 |

**pause가 막는 건 첫 줄 하나뿐이다.** 나머지가 Service의 영역.

```
Pod 삭제 → pause도 죽음 → refcount 0 → net_ns 소멸 → IP 회수 💀
```
**pause는 Pod와 함께 태어나고 죽으므로 Pod 밖의 안정성은 제공할 수 없다.**

| | pause | Service |
|---|---|---|
| 묶는 대상 | 컨테이너들 → **1개 Pod** | Pod들 → **1개 진입점** |
| 범위 | Pod 내부 | 클러스터 전체 |
| 수명 | Pod와 동일 | **Pod와 무관** |
| 정체 | **실제 프로세스** | iptables 규칙 + DNS (실체 없음) |
| 로드밸런싱 | 없음 | 있음 |

> **비유**: pause = **자동차 서스펜션**(노면 잔진동 흡수). Service = **내비게이션**(차가 바뀌어도 목적지는 그대로).

### 안정성 계층 전체

```
[DNS 이름]   payment-service      ← 영원히 안 변함
     ↓ CoreDNS
[Service IP] 10.96.0.42           ← Service 수명 동안 고정
     ↓ iptables DNAT (kube-proxy)
[Pod IP]     10.244.1.5           ← Pod 수명 동안 고정  ★ pause 담당
     ↓ 같은 net_ns
[컨테이너]   nginx, sidecar        ← 재시작해도 위 IP 유지
```

**각 층이 아래층의 흔들림을 흡수한다.** 겹치는 게 아니라 쌓이는 구조.

**Service만 있고 pause가 없으면**: 앱이 크래시할 때마다 Pod IP가 바뀌고 → Endpoints 갱신 → 모든 노드의 iptables 재작성 → 그 사이 트래픽 유실. **pause가 이 잔진동을 흡수한다.**

---

[← 서비스 디스커버리 — Service + CoreDNS](04-서비스디스커버리-service와-coredns.md) | [오브젝트 구조 — metadata / spec / status · Namespace · Label →](06-오브젝트구조-metadata-namespace-label.md)
