> [← kubernetes 목차](README.md)

# 계층 구조 — 컨테이너 / Pod / 노드 / 클러스터

## 계층 그림

```
┌─ 클러스터 (Cluster) ─────────────────────────┐
│                                              │
│  ┌─ 노드 A (물리/가상 머신) ─┐  ┌─ 노드 B ─┐ │
│  │  ┌─ Pod ─┐  ┌─ Pod ─┐    │  │ ┌─ Pod ─┐│ │
│  │  │ 컨테이너│  │ 컨테이너 │    │  │ │컨테이너││ │
│  │  │ 컨테이너│  │        │    │  │ └───────┘│ │
│  │  └───────┘  └───────┘    │  │           │ │
│  └──────────────────────────┘  └───────────┘ │
└──────────────────────────────────────────────┘
```

**컨테이너 ⊂ Pod ⊂ 노드 ⊂ 클러스터**

## ⚠️ 흔한 오해

> "클러스터가 Pod를 담는 그릇" → **중간에 노드가 빠졌다.**

- **노드는 진짜 컴퓨터** (EC2 인스턴스 같은 머신)
- **클러스터는 논리적 개념** — 노드들을 "하나처럼 쓰자"고 묶은 것
- 정확히는 **클러스터 = 노드들의 모임**이고, Pod는 노드들 위에 흩뿌려져 뜬다

| 층 | 정체 | 물리적? |
|---|---|---|
| 컨테이너 | 격리된 **프로세스** | 실체 있음 |
| Pod | 컨테이너 1개 이상 + **IP 1개** | 논리적 묶음 |
| 노드 | EC2 같은 **머신** | 실체 있음 |
| 클러스터 | 노드들의 모임 | 논리적 개념 |

## 왜 Pod라는 층이 굳이 있나?

**같은 Pod 안의 컨테이너들은 네트워크와 저장소를 공유한다.**

```
Pod (IP: 10.244.1.5)
├─ 컨테이너 A (Spring Boot, :8080)
└─ 컨테이너 B (로그 수집기)
   → B는 localhost:8080 으로 A에 접근 가능!
```

Pod당 **IP 하나**를 받고 그 안의 컨테이너들이 나눠 쓴다. 그래서 서로를 `localhost`로 부른다.

> **Java 비유**: 컨테이너가 각각의 객체라면, Pod는 **같은 JVM 안에서 힙을 공유하는 상태**. 다른 Pod는 완전히 다른 JVM 프로세스라 반드시 네트워크를 거쳐야 한다.

실무에선 **Pod 하나에 컨테이너 하나**가 대부분. 로그 수집기·프록시 같은 **사이드카**가 필요할 때만 2개 이상.

## ★ 계층별 구분 수단

| 계층 | 구분 수단 | 예시 | 유일성 범위 |
|---|---|---|---|
| 컨테이너 | **포트** | `:8080`, `:9090` | Pod 안에서 |
| Pod | **Pod IP** | `10.244.1.5` | **클러스터 전체** |
| 노드 | **노드 이름** (+ 노드 IP) | `node-01` / `192.168.0.11` | 클러스터 전체 |
| 클러스터 | **API 서버 주소** | `https://192.168.65.3:6443` | 전 세계 |

> **주의**: Pod IP는 노드 안이 아니라 **클러스터 전체에서 유일**하다. 노드가 달라도 절대 안 겹친다.

## 노드의 진짜 식별자는 IP가 아니라 "이름"

```bash
kubectl get nodes
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   5d    v1.36.2
```

IP는 재부팅하면 바뀔 수 있다(DHCP). 그래서 k8s는 **이름을 안정적 신원**으로 쓰고, IP는 Node 객체의 **속성 중 하나**로 들고 있다.

```bash
kubectl get nodes -o wide   # INTERNAL-IP 컬럼에 노드 IP
```

> **Java 비유**: `equals()`를 IP가 아니라 `name`으로 구현한 것. IP는 mutable field, name은 식별자.

---

## 3-1. ⚠️ 컨트롤 플레인은 계층이 아니라 "역할"이다

흔한 오해 하나를 먼저 정리한다.

```
❌ 클러스터 > 노드 > [컨트롤 플레인 + 워커 노드] > 컨테이너
⭕ 클러스터 > 노드 > Pod > 컨테이너
              ↑ 이 노드가 control-plane 역할이거나 worker 역할
```

**컨트롤 플레인 노드도, 워커 노드도 전부 그냥 "노드"다.** 똑같은 EC2 인스턴스일 수도 있다. 차이는 **그 위에서 뭐가 도느냐**뿐이다.

```bash
kubectl get nodes
# NAME             ROLES           ← ROLES 컬럼이 "역할"
# docker-desktop   control-plane

kubectl get nodes --show-labels | tr ',' '\n' | grep node-role
# node-role.kubernetes.io/control-plane=    ← 진짜로 그냥 라벨이다
```

> **Java 비유**: `Node` 클래스의 `role` 필드값 차이일 뿐, 별도 서브클래스가 아니다.

계층은 **클러스터 → 노드 → Pod → 컨테이너 4단계**이고, 컨트롤 플레인은 이 계층에 들어가지 않는다. (자세한 내용은 [클러스터 아키텍처](07-클러스터-아키텍처.md))

---

## 3-2. Pod 안에는 무엇이 있나

```
┌─ Pod ────────────────────────────────────┐
│  🔒 공유 공간                             │
│   ├─ IP 1개 (10.244.1.5)                 │
│   ├─ 포트 공간 1개 (1~65535)              │
│   └─ 볼륨 (파일 공유용)                   │
│                                          │
│  📦 컨테이너들                            │
│   ├─ pause 컨테이너 (숨겨진 껍데기)       │
│   ├─ init 컨테이너 (먼저 실행 후 종료)    │
│   ├─ 메인 컨테이너 (내 앱)                │
│   └─ 사이드카 컨테이너 (보조 역할)        │
└──────────────────────────────────────────┘
```

### ★ 공유되는 것 / 안 되는 것

| 항목 | 공유? | 결과 |
|---|---|---|
| **NET namespace** | ✅ | IP 하나, `localhost` 통신, **포트 충돌 가능** |
| **IPC namespace** | ✅ | 공유 메모리로 통신 가능 |
| **UTS namespace** | ✅ | hostname 동일 |
| **볼륨** | ✅ (마운트하면) | 파일 주고받기 |
| **MNT namespace** | ❌ | **파일시스템은 각자 따로!** |
| **PID namespace** | ❌ (기본값) | 서로의 프로세스 안 보임 |
| **cgroup** | ❌ | 컨테이너별로 따로 제한 |

> **⭐ 가장 헷갈리는 지점**: 네트워크는 공유하는데 **파일시스템은 안 한다.** 파일을 주고받으려면 **볼륨을 양쪽 다 명시적으로 마운트**해야 한다.
>
> **왜 이렇게 설계했나**: 파일시스템까지 공유하면 컨테이너마다 다른 베이스 이미지(alpine, ubuntu)를 못 쓴다. **이미지 독립성은 지키면서 네트워크만 합친** 절충안이다.

```bash
# 직접 확인 — localhost는 통하는데 파일은 안 보임
kubectl exec two -c helper -- wget -qO- localhost:80   # ✅ 통함
kubectl exec two -c web    -- ls /usr/share/nginx      # 있음
kubectl exec two -c helper -- ls /usr/share/nginx      # 없음!
```

---

## 3-3. pause 컨테이너 — 아무 일도 안 하는 주인공

```c
// pause의 전체 소스코드 (거의 이게 전부)
static void sigreap(int signo) {
    while (waitpid(-1, NULL, WNOHANG) > 0);   // 좀비 수거
}
int main() {
    signal(SIGCHLD, sigreap);
    for (;;) pause();                          // 신호 올 때까지 잠듦
}
```

**역할: NET namespace를 붙잡아두는 말뚝.**

```
① pause가 먼저 뜸 → NET namespace 생성 + IP 배정
② 앱 컨테이너가 그 namespace에 합류 (setns)
③ 사이드카도 합류
```

### 왜 필요한가 — pause 없으면 생기는 문제 4가지

| # | 문제 | 설명 |
|---|---|---|
| ① | **IP 변경** | 앱이 namespace 주인이면 앱 크래시 → refcount 0 → namespace 소멸 → 재시작 시 새 IP |
| ② | **주인 선정 불가** | 컨테이너 2개 이상일 때 누가 namespace를 만들지 정할 수 없음. 시작 순서가 곧 소유권이 됨 |
| ③ | **좀비 수거** | `shareProcessNamespace: true`일 때 PID 1이 고아 프로세스를 수거해야 함 |
| ④ | **준비 시점 불명확** | init 컨테이너도 네트워크가 필요한데, init이 namespace를 만들면 init 종료 시 사라짐 |

> **참조 카운트로 설명하면**: namespace는 가리키는 프로세스가 0개가 되면 소멸하는 커널 객체다(Java GC와 동일). pause가 **refcount를 1 이상으로 유지**해서 앱이 재시작해도 IP가 살아있는 것이다.

> **엄밀히는 필수가 아니다**: `ip netns add`처럼 bind mount로 NET namespace만 붙잡는 것도 가능하다. 다만 **PID namespace는 프로세스가 반드시 필요**하고, 어차피 프로세스를 띄울 거면 namespace 전체를 맡기는 게 깔끔하다.

### pause는 PID 1인가? — 기본값에선 아니다

```
[기본값: shareProcessNamespace: false] PID namespace 3개
pause  → PID ns #1, 자기가 1번
nginx  → PID ns #2, 자기가 1번    ← 각자 1번, 서로 안 보임
sidecar→ PID ns #3, 자기가 1번

[shareProcessNamespace: true] PID namespace 1개
pause (PID 1)  ← 진짜 1번
├─ nginx   (PID 6)
└─ sidecar (PID 12)
```

**기본 모드에선 앱이 PID 1이라 문제가 생긴다.**
- PID 1은 커널이 기본 시그널 핸들러를 안 붙여줌 → **SIGTERM 무시** → graceful shutdown 실패 → 30초 후 SIGKILL
- 앱이 만든 고아 프로세스가 좀비로 남음

**해결**: `tini` 같은 init 프로세스를 PID 1로 두거나, `shareProcessNamespace: true`로 pause에 맡긴다.
```dockerfile
ENTRYPOINT ["/tini", "--", "java", "-jar", "app.jar"]
```

### Pod 레벨 cgroup의 기준점

```
/sys/fs/cgroup/kubepods.slice/
  └── kubepods-...-pod<UID>.slice/          ← ★ Pod 레벨
      ├── memory.max = 640Mi                 (컨테이너 limits 합계 = 천장)
      ├── cri-containerd-<pause>.scope/
      ├── cri-containerd-<nginx>.scope/      memory.max = 512Mi
      └── cri-containerd-<sidecar>.scope/    memory.max = 128Mi
```

pause가 이 cgroup에 들어가서 **최소 1개 프로세스를 유지**한다(빈 cgroup은 정리될 수 있음). 그리고 Pod 레벨 한도가 **부모 천장**으로 작용한다 — [cgroup 중첩 구조](../container/03-cgroup-중첩과-jvm-oomkilled.md)와 같은 구조.

---

## 3-4. init 컨테이너 — 레이어가 아니라 순서

```
❌ 레이어 모델 (오해)
[init 컨테이너]  ← 베이스로 깔고
    ↓ 덮어쓰기
[메인 컨테이너]  ← 차이점만 얹음

⭕ 시간 모델 (실제)
시간 →
[init 실행]───[종료 💀]
                    [메인 실행]────계속 살아있음───→
```

| 규칙 | 내용 |
|---|---|
| **1. 완전히 끝나야 다음** | init이 `exit 0` 해야 메인이 시작 |
| **2. 순서 보장** | 여러 개면 위에서부터 하나씩 |
| **3. 실패 시 재시도** | `exit 1`이면 계속 재시작, 메인은 영원히 안 뜸 |
| **4. 파일시스템 독립** | init의 `/work`와 메인의 `/work`는 **서로 다른 폴더** |

**4번 때문에 볼륨이 필수다.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-web            # ← 그냥 Pod 이름. 베이스 이미지가 아니다.
spec:
  initContainers:
    - name: create-index
      image: busybox
      command: ['sh','-c','echo "<h1>Hello</h1>" > /work/index.html']
      volumeMounts:
        - name: work
          mountPath: /work                    # init이 보는 경로
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: work
          mountPath: /usr/share/nginx/html    # nginx가 보는 경로 (같은 볼륨)
  volumes:
    - name: work
      emptyDir: {}                            # Pod 소유의 빈 폴더
```

```
        [emptyDir: work]  ← 물리적으로 하나. Pod 소유.
              │
      ┌───────┴────────┐
      ▼                ▼
   /work          /usr/share/nginx/html
```

**시간 순서:**
```
① init 시작 → /work에 emptyDir 마운트 (빈 폴더)
② index.html 생성
③ init 종료 💀  ← 컨테이너는 죽지만 emptyDir은 Pod 소유라 생존 ★
④ nginx 시작 → 같은 emptyDir을 다른 경로에 마운트
⑤ index.html이 이미 거기 있음 → 서빙
```

> **비유**: 청소부(init)가 **공용 사물함**에 서류를 넣고 퇴근하고, 직원(nginx)이 출근해서 그 사물함을 연다. 두 사람은 마주친 적이 없지만 사물함을 통해 전달된다. 사물함이 없으면 청소부는 자기 가방에 서류를 넣고 그대로 퇴근한 셈이다.

### nginx 이미지의 원래 파일은?

nginx 이미지엔 원래 `/usr/share/nginx/html/index.html`이 있다. 그 경로에 볼륨을 마운트하면 **원래 내용은 가려진다** — 삭제가 아니라 **마운트가 경로를 덮은 것**이다. (책상 위 서류에 상자를 올려놓은 것과 같다)

### ⚠️ `containers`는 필수 필드다

```yaml
spec:
  template:
    spec: {}     # ← error: spec.template.spec.containers: Required value
```

**컨테이너 없는 Pod는 만들어질 수 없다.** Pod는 그릇 자체가 아니라 **컨테이너들이 공유 환경을 갖게 하는 장치**이므로, 공유할 주체가 없으면 성립하지 않는다.

### init vs 사이드카

| | init 컨테이너 | 사이드카 |
|---|---|---|
| 실행 시점 | 메인 **전에** | 메인과 **동시에** |
| 수명 | 끝나면 **죽음** | 계속 **살아있음** |
| 여러 개면 | **순차** | **병렬** |
| YAML | `initContainers:` | `containers:` (두 번째부터) |
| 용도 | DB 대기, 마이그레이션, 권한 설정 | 로그 수집, 프록시, 메트릭 |

**실무에서 가장 유용한 패턴 — DB 대기:**
```yaml
initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh','-c','until nc -z mysql-service 3306; do sleep 2; done']
```
Spring Boot가 DB 없이 뜨다 죽고 재시작을 반복하는 걸 막는다. 그리고 **`nc` 같은 도구를 앱 이미지에 넣지 않아도 된다** — 도구와 앱의 분리가 init 컨테이너의 큰 장점이다.

---

[네트워크 — 오버레이와 언더레이 →](02-네트워크-오버레이와-언더레이.md)
