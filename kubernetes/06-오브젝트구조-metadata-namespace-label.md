> [← kubernetes 목차](README.md)

# 오브젝트 구조 — metadata / spec / status · Namespace · Label

## 9-1. 모든 k8s 오브젝트는 3부 구성

```yaml
apiVersion: v1
kind: Pod
metadata:      # ① 나는 누구인가 — 신원
  name: web
  namespace: default
  labels:
    app: web
spec:          # ② 나는 어떠해야 하는가 — 원하는 상태 (사용자가 작성)
  containers: [...]
status:        # ③ 나는 지금 어떤가 — 실제 상태 (k8s가 작성)
  phase: Running
  podIP: 10.244.1.5
```

| 섹션 | 의미 | 누가 씀 | 형식 |
|---|---|---|---|
| **metadata** | **신원** — 이름표, 소속 | 사용자 | ⭕ 모든 오브젝트 공통 |
| **spec** | **의도** — 원하는 상태 | 사용자 | ❌ 종류마다 다름 |
| **status** | **현실** — 실제 상태 | **k8s** | ❌ 종류마다 다름 |

> **Java 비유**
> ```java
> abstract class K8sObject { Metadata metadata; }   // 공통 (부모 클래스)
> class Pod     extends K8sObject { PodSpec spec; PodStatus status; }
> class Service extends K8sObject { ServiceSpec spec; ... }
> ```

### ★ 왜 labels는 metadata, selector는 spec인가

```
labels   = "나는 이런 이름표를 달고 있다"   → 자기 신원   → metadata
selector = "나는 이런 애들을 관리하고 싶다" → 원하는 상태 → spec
```

**labels는 자기 자신에 대한 서술, selector는 외부 대상에 대한 요구.**

> 명찰은 신분증에, 찾는 조건은 업무 지시서에 적히는 게 자연스럽다.

이 배치 덕분에 종류를 안 가리는 검색이 가능하다.
```bash
kubectl get all -l app=web    # Pod, Service, Deployment 전부
```
라벨이 metadata에 있어 API Server가 **오브젝트 종류와 무관하게 인덱싱**할 수 있기 때문이다.

---

## 9-2. Label — 붙이는 쪽과 찾는 쪽은 항상 짝이다

```
[붙이는 쪽]  metadata.labels    ← 라벨을 "단다"
[찾는 쪽]    spec.selector      ← 그 라벨을 "검색한다"
```

**`selector`가 있다는 건 어딘가에 `labels`가 있다는 뜻이다.** 짝이 없으면 아무것도 못 찾는다.

| 라벨의 주인 | 붙는 위치 | 찾는 쪽 | 문법 |
|---|---|---|---|
| **Pod** | `spec.template.metadata.labels` | Service | `spec.selector` |
| **Pod** | 위와 같음 | ReplicaSet | `spec.selector.matchLabels` |
| **Node** | 노드 오브젝트 | Pod | `spec.nodeSelector` |
| **Namespace** | Namespace 오브젝트 | NetworkPolicy | `namespaceSelector` |

### ⚠️ Deployment에 라벨이 세 군데 나오는 이유

```yaml
kind: Deployment
metadata:
  labels:
    app: web                 # ① Deployment 자신의 명찰 (선택사항)
spec:
  selector:
    matchLabels:
      app: web               # ② 검색 조건 (필수, 변경 불가)
  template:
    metadata:
      labels:
        app: web             # ③ ★ 실제로 Pod에 붙는 라벨
```

**②와 ③은 반드시 일치해야 한다.** 불일치하면 k8s가 거부한다.
```
error: `selector` does not match template `labels`
```
(일치하지 않으면 "만들어도 안 세어짐 → 또 만듦" 무한 반복이 되기 때문)

```
Deployment.selector ──찾음──> Pod의 labels <──찾음── Service.selector
                                  ↑ 여기가 유일한 실체
```

**Deployment와 Service는 서로를 모른다.** 둘 다 Pod의 라벨을 독립적으로 볼 뿐이다.

### 셀렉터 조건은 AND

```yaml
selector:
  app: web
  tier: frontend     # "app=web 그리고 tier=frontend"
```

| Pod | 결과 |
|---|---|
| `app=web, tier=frontend` | ✅ |
| `app=web, tier=backend` | ❌ |
| `app=web` | ❌ (tier 없음) |

**조건을 추가할수록 범위가 좁아진다.** OR은 `matchExpressions`에서만 가능하다.

```yaml
# Service — 등호만 (equality-based)
spec:
  selector:
    app: web

# Deployment/ReplicaSet — 표현식 가능 (set-based)
spec:
  selector:
    matchLabels:
      app: web
    matchExpressions:
      - key: version
        operator: In
        values: [v1, v2]        # ← OR
```

```bash
kubectl get pods -l 'app=web,version in (v1,v2)'
kubectl get pods -l '!version'          # version 라벨이 없는 것
```

**실무에선 용도별로 조합한다.**
```yaml
labels:
  app: payment       # 어떤 서비스
  tier: backend      # 어느 계층
  env: prod          # 어느 환경
  version: v2        # 어느 버전
```
같은 Pod 집합을 **여러 각도로 슬라이싱**할 수 있는 게 라벨의 힘이다. (카나리 배포: `app=payment, version=v2`만 선택)

---

## 9-3. Namespace vs Label — 폴더 vs 태그

| | Namespace | Label |
|---|---|---|
| **개수** | 오브젝트당 **1개** | **여러 개** |
| **변경** | ❌ **불가능** (immutable) | ✅ 언제든 |
| **역할** | **어디 소속인가** | **어떤 특성인가** |
| 이름 유일성 | Namespace 안에서 유일 | 무관 |
| RBAC/Quota | ⭕ 걸 수 있음 | ❌ |
| 삭제 시 | **안의 모든 것이 삭제** | 꼬리표만 제거 |

### Namespace는 "주소의 일부"라서 하나뿐이다

```
etcd 경로:  /registry/pods/production/my-app
                          └ns┘      └name┘
```

파일이 두 폴더에 동시에 있을 수 없듯, Pod도 두 Namespace에 못 있는다. **그래서 변경도 불가능하다** — 지우고 새로 만들어야 한다.

### name과 namespace의 차이

```
metadata.namespace  →  폴더
metadata.name       →  파일 이름
둘을 합쳐야         →  전체 주소
```

```bash
kubectl get pod my-app                # 실은 -n default 가 생략된 것
kubectl get pod my-app -n production  # 완전한 형태
```

### Namespace가 필요한 이유 3가지

**① 이름 재사용** — 없으면 모든 오브젝트에 환경 접두사를 박아야 한다.
```
[Namespace 없이]           [Namespace 있으면]
dev-my-app                 my-app  (-n dev)
staging-my-app             my-app  (-n staging)
prod-my-app                my-app  (-n prod)
dev-my-app-service         ← YAML 하나로 3개 환경 배포 ✅
staging-my-app-service
... 💀
```

**② RBAC 권한 경계** — 권한은 네임스페이스 단위로만 걸 수 있다.
```yaml
kind: RoleBinding
metadata:
  namespace: dev      # dev에서만 유효
```
Namespace가 없으면 `resourceNames`에 오브젝트를 일일이 열거해야 한다.

**③ ResourceQuota** — 자원 총량 제한.
```yaml
kind: ResourceQuota
metadata:
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    pods: "20"
```

### 함께 작동하는 방식

**Service의 selector는 같은 Namespace 안에서만 Label을 찾는다.**

> **SQL 비유**
> ```sql
> SELECT * FROM pods
> WHERE namespace = 'prod'         -- Namespace: 검색 범위
>   AND labels->>'app' = 'web'     -- Label: 검색 조건
> ```

---

## 9-4. ⚠️ k8s Namespace vs Linux namespace — 이름만 같다

| | Linux namespace | Kubernetes Namespace |
|---|---|---|
| **정체** | 커널 자료구조 | **etcd에 저장된 문자열 필드** |
| **적용 레벨** | **프로세스** | **k8s 오브젝트** |
| **목적** | 프로세스 격리 | 이름 충돌 방지 + 권한/할당량 구획 |
| **격리 수준** | 진짜 격리 (안 보임) | **기본적으로 격리 아님** ⚠️ |
| 누가 앎 | 커널 | k8s만 (커널은 모름) |

> **Java 비유**
> - **Linux namespace** = `ThreadLocal` (런타임 격리)
> - **k8s Namespace** = **패키지 이름** `com.a.UserService` vs `com.b.UserService`

### 가장 중요한 오해: 네트워크가 격리되지 않는다

```bash
kubectl exec -n default my-pod -- curl http://svc.production.svc.cluster.local
# → 통신됨! ✅
```

**k8s Namespace는 이름만 나눌 뿐 트래픽을 막지 않는다.** 막으려면 **NetworkPolicy**를 별도로 설정해야 한다.

```
[k8s Namespace: production]   ← etcd의 논리적 구획
  └─ Pod: my-app              ← k8s 오브젝트
      └─ 노드에서 실행되면
          └─ [Linux namespace] ← 커널 레벨, 여기서 진짜 격리
```

**층이 완전히 다르다.** k8s Namespace를 바꿔도 Linux namespace 구성은 하나도 안 바뀐다.

### Namespaced vs Cluster-scoped 오브젝트

| Namespaced (소속됨) | Cluster-scoped (소속 없음) |
|---|---|
| Pod, Deployment, ReplicaSet, Job | **Node** |
| Service, Ingress, Endpoints | **Namespace** 자신 |
| ConfigMap, Secret | **PersistentVolume (PV)** |
| **PersistentVolumeClaim (PVC)** | StorageClass |
| ServiceAccount, Role, RoleBinding | ClusterRole, ClusterRoleBinding |
| NetworkPolicy, ResourceQuota | CustomResourceDefinition |

**구분 기준**: "특정 팀/환경 소유인가, 클러스터 공용 인프라인가"

> PV/PVC 쌍이 좋은 예시다. **PV(실제 디스크) = 클러스터 자원**, **PVC(사용 요청) = 팀 소유**.

```bash
kubectl api-resources --namespaced=true    # 네임스페이스 있는 것
kubectl api-resources --namespaced=false   # 없는 것
```

---

[← 워크로드 오브젝트 — Deployment / ReplicaSet / Service](05-워크로드-deployment-replicaset-service.md) | [클러스터 아키텍처 — 누가 실제로 실행하는가 →](07-클러스터-아키텍처.md)
