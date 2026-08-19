# 쿠버네티스 기초 정리

> 리눅스 커널 밑바닥부터 쿠버네티스 오브젝트까지 — 기초 → 심화 순서
>
> 다루는 범위: 커널의 정체 / namespace·cgroup 원리 / 컨테이너의 실체 / k8s 계층 구조 / 네트워크 2층 구조 / kubectl context / 워크로드 오브젝트 / 설정 분리 / Ingress

---

## 목차

- [서장. 리눅스 커널 — 모든 것의 바닥](#서장-리눅스-커널--모든-것의-바닥)
- [0장. 전제 지식 — 컨테이너는 VM이 아니다](#0장-전제-지식--컨테이너는-vm이-아니다)
- [1장. 쉘 문법 — heredoc과 tee](#1장-쉘-문법--heredoc과-tee)
- [2장. k8s 설치 전 커널 설정 4단계](#2장-k8s-설치-전-커널-설정-4단계)
- [3장. k8s 계층 구조 — 컨테이너 / Pod / 노드 / 클러스터](#3장-k8s-계층-구조--컨테이너--pod--노드--클러스터)
- [4장. 네트워크 — 오버레이와 언더레이](#4장-네트워크--오버레이와-언더레이)
- [5장. kubectl context와 kubeconfig](#5장-kubectl-context와-kubeconfig)
- [6장. cgroup 중첩 구조와 JVM OOMKilled](#6장-cgroup-중첩-구조와-jvm-oomkilled)
- [7장. 서비스 디스커버리 — Service + CoreDNS](#7장-서비스-디스커버리--service--coredns)
- [8장. 워크로드 오브젝트 — Deployment / ReplicaSet / Service](#8장-워크로드-오브젝트--deployment--replicaset--service)
- [9장. 오브젝트 구조 — metadata / spec / status, Namespace, Label](#9장-오브젝트-구조--metadata--spec--status-namespace-label)
- [10장. 클러스터 아키텍처 — 누가 실제로 실행하는가](#10장-클러스터-아키텍처--누가-실제로-실행하는가)
- [11장. 설정 분리 — ConfigMap / Secret](#11장-설정-분리--configmap--secret)
- [12장. Ingress — 클러스터의 L7 대문](#12장-ingress--클러스터의-l7-대문)
- [13장. 전체 연결 지도](#13장-전체-연결-지도)
- [14장. 면접 대비 Q&A](#14장-면접-대비-qa)
- [15장. 실습 체크리스트](#15장-실습-체크리스트)

---

## 서장. 리눅스 커널 — 모든 것의 바닥

> 이 장은 나머지 전부의 전제다. namespace, cgroup, 컨테이너, Pod가 감이 안 잡히면 여기가 비어 있는 것이다.

### S-1. 컴퓨터는 3층 건물이다

```
┌─────────────────────────────────┐
│  프로그램 (내 Spring Boot 앱)    │  ← 3층
├─────────────────────────────────┤
│  커널 (리눅스)                   │  ← 2층
├─────────────────────────────────┤
│  하드웨어 (CPU, RAM, 디스크, 랜) │  ← 1층
└─────────────────────────────────┘
```

> **딱 하나만 기억할 것: 3층은 1층을 직접 만질 수 없다. 반드시 2층에 부탁해야 한다.**

Java에서 `new FileWriter("a.txt")` 를 해도 프로그램이 디스크 헤드를 움직이는 게 아니다. **커널에게 "이거 좀 써줘"라고 부탁**하는 것이다.

#### 왜 막아놨나

프로그램이 하드웨어를 직접 만질 수 있으면 남의 앱 메모리를 읽고(비밀번호 탈취), 버그 하나로 디스크를 날리고, CPU를 독점한다.

**그래서 CPU에 물리적 장치를 넣었다 — Ring.**

```
Ring 0 = 커널 모드   → 하드웨어 명령 실행 가능
Ring 3 = 사용자 모드 → 시도하면 CPU가 즉시 차단
```

내 프로그램은 **평생 Ring 3에서 산다.** (0-4장 VM 격리에서 이 Ring이 다시 등장한다)

---

### S-2. 커널이 하는 일은 4가지

커널은 **하드웨어 4종류를 대신 관리해주는 존재**다.

| 관리 대상 | 커널이 하는 일 | 내가 쓰는 것 |
|---|---|---|
| **CPU** | 프로세스 스케줄링 (누가 언제 실행되나) | 프로그램 실행 |
| **메모리** | 어느 프로그램에 어느 영역을 줄지 | `new`, `malloc` |
| **디스크** | 파일시스템 (파일이라는 개념 자체) | 파일 읽기/쓰기 |
| **네트워크 카드** | TCP/IP 처리, 패킷 송수신 | 소켓 통신 |

> **Spring 비유**: 커널은 **거대한 IoC 컨테이너**다. 내가 직접 객체를 만들지 않고 Spring에 요청하듯, 내가 직접 하드웨어를 만지지 않고 커널에 요청한다. Spring이 빈의 생명주기를 관리하듯 커널은 프로세스·메모리·파일의 생명주기를 관리한다.

---

### S-3. 부탁하는 법 = 시스템 콜 (syscall)

3층이 2층에 부탁하는 **유일한 창구**. 커널의 **공개 API**다.

```java
FileWriter fw = new FileWriter("a.txt");
fw.write("hello");
```
↓ 실제로는
```
JVM → C 라이브러리 → syscall: open("a.txt", O_WRONLY)   ← 커널 진입
                     syscall: write(fd, "hello", 5)     ← 커널 진입
```

syscall이 호출되는 순간 **CPU가 Ring 3 → Ring 0으로 전환**된다. 처리 후 다시 Ring 3으로 복귀.

| syscall | 하는 일 |
|---|---|
| `fork()` / `clone()` | 새 프로세스 만들기 |
| `execve()` | 프로그램 실행 |
| `open` / `read` / `write` | 파일 조작 |
| `socket` / `bind` / `listen` | 네트워크 |
| `mmap` | 메모리 할당 |

> **★ "리눅스 커널 기능"이란** = 커널이 제공하는 **syscall과 그 뒤의 관리 로직**. namespace도 cgroup도 전부 여기 속한다.

```bash
strace ls    # ls가 부르는 syscall을 전부 출력
```

---

### S-4. 프로세스란 무엇인가 (커널 관점)

우리는 프로세스를 "실행 중인 프로그램"이라 알지만, **커널 입장에선 자료구조 하나**다.

```c
// 리눅스 커널 실제 코드 (단순화)
struct task_struct {
    pid_t   pid;                   // 프로세스 번호
    char    comm[16];              // 이름
    struct  mm_struct *mm;         // 메모리 정보
    struct  files_struct *files;   // 열어둔 파일들
    struct  nsproxy *nsproxy;      // ★ 내가 속한 namespace들
    struct  css_set *cgroups;      // ★ 내가 속한 cgroup들
};
```

커널은 이 구조체를 리스트로 들고 있다.

```
[systemd(1)] → [sshd(890)] → [java(45231)] → [nginx(45890)] → ...
```

`ps aux`는 **커널이 이 리스트를 훑어서 보여주는 것**이다.

> **Java 비유**
> ```java
> class Kernel {
>     List<Process> allProcesses;       // 모든 프로세스
>     List<Socket> allSockets;          // 모든 네트워크 연결
>     Map<Integer, Integer> portTable;  // 포트 사용 현황
>     MountTable mounts;                // 마운트된 파일시스템
> }
> ```
> 커널이 관리하는 **전역 목록**들이 있다.

---

### S-5. ★ 그래서 namespace가 왜 필요한가

문제는 **이 목록들이 전부 하나뿐**이라는 것.

```
전역 프로세스 목록  1개  → 모두가 남의 프로세스를 봄
전역 포트 테이블   1개  → 8080은 세상에 하나뿐
전역 마운트 테이블  1개  → 모두가 같은 / 를 봄
```

한 서버에 앱 두 개를 띄우면 충돌하는 근본 원인이다. **공유 자원이 하나뿐이라서.**

#### 해결책 두 가지

| 방법 | 내용 | 결과물 |
|---|---|---|
| **A** | 컴퓨터를 하나 더 산다 → 커널이 하나 더 → 목록도 하나 더 | **VM** |
| **B** | 커널 안의 목록만 여러 벌 만든다 | **namespace** |

```
[기존]                      [namespace 도입 후]
커널                         커널
 └─ 프로세스 목록 1개         ├─ 프로세스 목록 #1  ← 호스트가 봄
    (모두가 공유)             ├─ 프로세스 목록 #2  ← 컨테이너 A가 봄
                             └─ 프로세스 목록 #3  ← 컨테이너 B가 봄
```

> **namespace = 커널이 관리하는 전역 목록을 여러 벌로 복제하는 기능**

> **★ Java 비유 (본질)**
> ```java
> // 기존: static 필드 (전역에 하나)
> class Kernel {
>     static List<Process> processes;   // 모두가 같은 걸 봄
> }
>
> // namespace: 인스턴스 필드로 바꾼 것
> class Namespace {
>     List<Process> processes;          // 인스턴스마다 따로
> }
> Namespace host       = new Namespace();
> Namespace containerA = new Namespace();
> ```
> **"static을 인스턴스로 바꾼 것"** — 이것이 namespace의 정체다.

#### "시야 차단"이라는 표현의 진짜 의미

컨테이너 A의 프로세스는 **목록 #2만 참조**한다. 목록 #1이 존재하는지도 모른다.

```bash
docker run --rm alpine ps aux
# PID 1  ← 목록 #2에는 이것뿐이라 1번
```

**숨긴 게 아니라, 애초에 다른 목록을 보고 있는 것.**

#### 왜 7종류인가

커널이 관리하는 전역 목록이 한 종류가 아니기 때문. **목록 종류마다 namespace가 하나씩** 있다.

| 커널의 전역 자원 | 복제하는 namespace |
|---|---|
| 프로세스 목록 | **PID** |
| 네트워크 스택 (IP·포트·라우팅·iptables) | **NET** |
| 마운트 테이블 (파일시스템) | **MNT** |
| 호스트명 | **UTS** |
| 공유메모리·세마포어 | **IPC** |
| 유저/그룹 ID 테이블 | **USER** |
| cgroup 트리 뷰 | **CGROUP** |

그리고 **골라서 쓸 수 있다.**
```bash
docker run --pid=host alpine ps aux
# PID namespace만 안 만듦 → 호스트 프로세스가 다 보임
# 나머지(NET, MNT...)는 여전히 격리됨
```

---

### S-6. cgroup — 이름부터

> **cgroup = Control Group = 제어 그룹**
> `Control(제어) + Group(묶음)` = **"프로세스를 묶어서 자원 사용을 제어한다"**

#### namespace로는 안 되는 것

namespace는 **보이는 것을 자를 뿐** 자원을 못 막는다.

```
컨테이너 A: 자기 프로세스만 보임 (namespace ✅)
          하지만 메모리를 8GB 다 먹을 수 있음 ❌
          → 컨테이너 B가 OOM으로 죽음
```

**시야를 잘라도 물리 자원은 여전히 하나를 나눠 쓴다.** 그래서 별도 기능이 필요했다.

#### 작동 방식 — 전부 파일 조작

```bash
mkdir /sys/fs/cgroup/mygroup                          # 1. 그룹 생성
echo 536870912 > /sys/fs/cgroup/mygroup/memory.max    # 2. 제한값 기록 (512MB)
echo 45231     > /sys/fs/cgroup/mygroup/cgroup.procs  # 3. 프로세스 편입
                                                      # 4. 커널이 강제
```

리눅스 철학이 **"모든 것은 파일"** 이라서 제한값도 파일에 적는다. 그래서 **"cgroup 경로"** 라는 개념이 생긴다.

```
/sys/fs/cgroup/
└── docker/                     memory.max = 4GB      ← 부모 한도
    ├── a1b2c3.../              memory.max = 512MB    ← 컨테이너 A
    └── d4e5f6.../              memory.max = 256MB    ← 컨테이너 B
```

**트리인 이유**: 부모 한도가 자식들을 덮기 위해서. 자식 합이 5GB여도 부모가 4GB면 4GB에서 막힌다. (6장의 "VM 8GB 제한이 이긴다"가 이 원리)

#### 그럼 CGROUP namespace는?

```
cgroup            = 제한 기능 (커널 기능 B)
CGROUP namespace  = cgroup 트리 "뷰"를 복제 (커널 기능 A의 7번째)
```

cgroup 트리도 커널의 전역 자원이므로 namespace로 복제 대상이 된다. **이름만 겹칠 뿐 계층 관계가 아니다.**

```bash
# 호스트 시점
cat /proc/45231/cgroup      # 0::/docker/a1b2c3.../   ← 전체 경로 노출
# 컨테이너 시점
docker exec t cat /proc/self/cgroup   # 0::/          ← 자기가 루트인 척
```
PID namespace가 자기를 1번으로 보이게 하는 것과 **완전히 같은 방식**. 표시만 바꾸고 **제한은 그대로**다.

---

### S-7. namespace와 cgroup은 완전히 별개다

```
┌─────────────── 리눅스 커널 ───────────────┐
│                                           │
│  [기능 A] namespace  — 시야 차단           │
│    ├─ PID    ├─ NET   ├─ MNT              │
│    ├─ UTS    ├─ IPC   ├─ USER             │
│    └─ CGROUP  ← cgroup을 "대상으로" 삼음   │
│                                           │
│  [기능 B] cgroup     — 자원 제한           │
│    ├─ memory  ├─ cpu   ├─ pids  ├─ io     │
│                                           │
└───────────────────────────────────────────┘
```

| | namespace | cgroup |
|---|---|---|
| 질문 | **"뭐가 보여?"** | **"얼마나 쓸 수 있어?"** |
| 동사 | 가린다 | 막는다 |
| 커널 동작 | 목록을 복제 | 사용량을 카운트 |
| 없으면 | 남의 것이 다 보임 | 자원 독식 |
| 등장 | 2002~2016 (7개 순차) | 2007 (구글이 개발) |

**만들어진 시기도 개발자도 목적도 다르다.** 나중에 도커가 둘을 조합했을 뿐이다.

---

### S-8. /proc vs /sys — 신분증 vs 조종실

둘 다 **가상 파일시스템**이다. 디스크에 실제 파일이 있는 게 아니라 커널이 즉석에서 만들어낸다.

| | `/proc` (procfs) | `/sys` (sysfs) |
|---|---|---|
| 주인공 | **프로세스** | **커널 객체·장치** |
| 답하는 질문 | "**나**는 어떤 상태인가?" | "이 **기능**을 어떻게 조작하나?" |
| 쓰기 | 대부분 읽기 전용 | **쓰기 가능** |
| 비유 | **신분증** | **조종실 계기판** |

```bash
/proc/self/cgroup          # "나는 어느 cgroup 소속인가?"   ← 소속 조회
/sys/fs/cgroup/memory.max  # "이 cgroup의 상한은?"          ← 값 조회/변경
```

- `/proc/self/*` — status, environ, fd/, **ns/**, cgroup … 전부 **"나에 대한 정보"**
- `/sys/*` — fs/cgroup/, class/net/, block/, kernel/ … 전부 **"커널이 관리하는 대상"**

> `/proc/sys/` 는 `/proc` 안에 있지만 성격은 `/sys`에 가깝다. sysfs가 나중에 생기면서 남은 **역사적 잔재**이며, `sysctl`이 만지는 곳이 여기다. (2장)

#### ★ 이 차이가 JVM OOMKilled의 원인이다

도커는 컨테이너의 cgroup 디렉토리를 `/sys/fs/cgroup`으로 **바인드 마운트**해준다 (MNT namespace). 하지만 **`/proc/meminfo`는 가려주지 않는다.**

```bash
docker run --rm --memory=512m alpine sh -c '
  cat /sys/fs/cgroup/memory.max   # 536870912  = 512MB (정확)
  head -1 /proc/meminfo           # 8GB        = 호스트 것 (부정확) ★
'
```

JVM이 하필 뒤엣것을 읽어 힙을 2GB로 잡다가 죽었다. `-XX:+UseContainerSupport`가 하는 일이 **읽는 경로를 `/proc`에서 `/sys/fs/cgroup`으로 바꾸는 것**이다. (6장)

---

### S-9. 컨테이너 = 조립품

이제 컨테이너의 정체가 보인다. **새로운 기술이 아니다.**

```
컨테이너 = namespace(격리)
         + cgroup(자원 제한)
         + overlayfs(이미지 레이어)
         + capabilities(권한 축소)
         + seccomp(syscall 필터)
```

**이미 있던 커널 기능들의 조합.** 도커가 한 일은 이걸 `docker run` 한 줄로 포장한 것이다.

#### 도커 없이 손으로 만들어보기

```bash
sudo unshare --pid --net --mount --uts --fork bash

# 이 셸 안에서
hostname mycontainer    # UTS namespace → 호스트에 영향 없음
ps aux                  # PID namespace → 자기 것만 보임
ip addr                 # NET namespace → lo만 있음
exit
```
> **`unshare` 하나로 namespace를 만들 수 있다.** 이걸 해보면 "컨테이너는 마법이 아니다"가 체감된다.

---

### S-10. 이 문서의 개념들이 커널 어디에 붙는가

| 개념 | 커널상의 위치 | 등장 장 |
|---|---|---|
| `modprobe overlay` | **커널 모듈** = 기능을 나중에 끼우는 것 | 2장 |
| `br_netfilter` | 커널 모듈 (netfilter 관련) | 2장 |
| `sysctl` | 커널 **동작 스위치** (`/proc/sys/`) | 2장 |
| Pod가 IP 공유 | **NET namespace 공유** | 0-2, 3장 |
| 포트 충돌 없음 | NET namespace가 포트 테이블 복제 | 0-2장 |
| 컨테이너별 메모리 제한 | **cgroup** | 0-3장 |
| JVM OOMKilled | cgroup은 제한하나 `/proc`은 안 가림 | 6장 |
| VM 격리 | **하이퍼바이저 + CPU Ring/EPT** | 0-4장 |

---

## 0장. 전제 지식 — 컨테이너는 VM이 아니다

이 문서 전체에서 가장 중요한 전제.

### 커널 = 하드웨어를 직접 만지는 유일한 존재

```
[내 프로그램] → 부탁 → [커널] → 직접 조작 → [하드웨어]
```

### 컨테이너 = 격리된 "프로세스"

컨테이너는 가벼운 VM이 **아니다**. 리눅스 커널의 두 기능으로 만들어진 속임수다.

| 기능 | 역할 | 비유 |
|---|---|---|
| **namespace** | 시야 차단 — "네 눈엔 이 프로세스/네트워크만 보여" | 칸막이 |
| **cgroup** | 자원 제한 — "너는 CPU 1개, RAM 512MB만" | 예산 배정 |

### VM과의 근본적 차이

| | 가상머신(VM) | 컨테이너 |
|---|---|---|
| 커널 | **자기 커널을 따로 가짐** | **호스트 커널을 같이 씀** |
| 격리 담당 | 하이퍼바이저 (KVM, Hyper-V) | 커널 (namespace) |
| 격리 수준 | **하드웨어** (Intel VT-x, EPT/NPT) | 소프트웨어 (커널이 시야를 가림) |
| 정체 | 가짜 컴퓨터 | 격리된 프로세스 |
| 부팅 | 수십 초 | 즉시 |
| 뚫리면 | 매우 어려움 | 커널 취약점 → 호스트 장악 가능 |

### VM은 namespace/cgroup을 쓰나?

**"VM을 격리하는 수단"으로는 안 쓴다. 하지만 두 군데에 다 존재한다.**

```
namespace = 시야 격리 → VM 격리엔 안 씀 (하이퍼바이저가 대신함)
cgroup    = 자원 제한 → VM에도 씀! (호스트가 VM 프로세스에 걸어둠)
```

- **VM 안쪽**: 게스트가 리눅스면 완전한 커널이므로 namespace/cgroup 다 있음 → **그래서 VM 안에서 Docker를 돌릴 수 있다**
- **VM 바깥쪽**: 호스트 입장에서 VM은 그냥 프로세스(`qemu-system-x86_64`) → cgroup으로 자원 제한 가능

### ⚠️ 가장 중요한 귀결

> 컨테이너는 **호스트 커널을 그대로 쓴다.**
> → 호스트 커널 설정이 안 맞으면 컨테이너가 제대로 못 돈다.
> → **그래서 k8s 설치 전에 커널을 만져야 한다.** (2장)

VM이었다면 호스트 커널 설정 따위 상관없었을 것이다.

---

### 0-1. 격리(isolation)란 무엇인가

> **격리 = "너는 이것만 볼 수 있고, 이것만 만질 수 있어" 라고 범위를 잘라주는 것.**

핵심은 **"보인다 / 안 보인다"**. 물리적으로 떼어놓는 게 아니라 **시야를 자르는 것**이다.

#### 격리가 없으면 생기는 충돌

한 서버에 앱 두 개를 그냥 띄우면:

| 충돌 | 상황 | 담당 격리 |
|---|---|---|
| **포트 충돌** | 둘 다 8080을 쓰려다 하나가 못 뜸 | NET namespace |
| **라이브러리 충돌** | Java 8 vs 17 | MNT namespace |
| **파일 충돌** | 둘 다 `/tmp/data`에 씀 | MNT namespace |
| **프로세스 간섭** | `killall java` → 남의 앱도 죽음 | PID namespace |
| **자원 독식** | A가 메모리 다 먹어 B가 OOM | cgroup |

---

### 0-2. namespace — 시야 격리 (7종류)

리눅스 커널은 **자원의 종류별로** 시야를 따로 자를 수 있다.

| namespace | 자르는 대상 | 격리 효과 |
|---|---|---|
| **PID** | 프로세스 목록 | 남의 프로세스가 안 보임 |
| **NET** | 네트워크 (IP, 포트, 라우팅) | 8080을 각자 독점 |
| **MNT** | 파일시스템 마운트 | 각자 다른 `/` 를 봄 |
| **UTS** | 호스트명 | 각자 다른 hostname |
| **IPC** | 프로세스 간 통신 | 공유 메모리 분리 |
| **USER** | 유저/그룹 ID | 컨테이너 안 root ≠ 호스트 root |
| **CGROUP** | cgroup 뷰 | 자기 cgroup만 보임 |

#### PID namespace — 하나의 프로세스에 이름표 2개

```bash
docker run --rm alpine ps aux
# PID   USER   COMMAND
#   1   root   ps aux        ← 컨테이너 안에선 1번

# 호스트에서 같은 프로세스
ps aux | grep nginx
# PID 45231                  ← 완전히 다른 번호
```

**프로세스가 복사된 게 아니다. 하나의 프로세스에 두 개의 이름표가 붙은 것.**

> **비유**: 회사 **사번**과 **주민번호**. 같은 사람인데 맥락에 따라 다른 번호로 불린다.

```bash
kill -9 -1    # 컨테이너 안에서 실행 → 컨테이너 안의 것만 죽음
```
**볼 수 없으면 죽일 수도 없다.** 시야 차단이 곧 보호막.

#### ★ NET namespace — 가장 오해하기 쉬운 부분

```bash
docker run -d -p 8080:80 nginx   # 컨테이너 A
docker run -d -p 8081:80 nginx   # 컨테이너 B
```

**`-p 8080:80` 읽는 법:**
```
-p  8080  :  80
    ↑        ↑
  호스트    컨테이너
```
**왼쪽은 호스트 소속, 오른쪽은 컨테이너 소속.** 완전히 다른 공간의 포트 번호다.

```
┌─ 호스트 NET namespace ────────────────────────┐
│  IP: 192.168.0.10 / 포트공간 1~65535          │
│  ├─ 8080 사용 중 (docker가 잡음)              │
│  └─ 8081 사용 중                              │
│                                               │
│   ┌─ 컨테이너 A ─┐   ┌─ 컨테이너 B ─┐         │
│   │ 172.17.0.2   │   │ 172.17.0.3   │         │
│   │ 포트공간 별개 │   │ 포트공간 별개 │         │
│   │ └─ 80        │   │ └─ 80        │         │
│   └──────────────┘   └──────────────┘         │
└───────────────────────────────────────────────┘
```

**⚠️ 흔한 오해 교정**

> ❌ "NET namespace = 8080, 8081" (namespace를 **값**으로 이해)
> ⭕ "NET namespace = 포트 번호를 담는 **통 자체**"

```
namespace = [IP + 포트공간 1~65535 + 라우팅테이블 + iptables + 인터페이스]
            └───────────── 이 세트 전체가 하나의 namespace ─────────────┘
```

> **Java 비유 (정확함)**
> ```java
> class NetworkStack {
>     String ip;
>     Port[] ports = new Port[65536];
>     RoutingTable routes;
>     IptablesRules rules;
> }
> NetworkStack host = new NetworkStack();
> NetworkStack a    = new NetworkStack();   // 컨테이너 A
> NetworkStack b    = new NetworkStack();   // 컨테이너 B
> ```
> **`a.ports[80]`과 `b.ports[80]`은 다른 객체의 다른 필드.** 이름만 같을 뿐.
> namespace = **인스턴스**, 포트 번호 = 그 인스턴스 안의 **배열 인덱스**.

**A와 B의 namespace가 다르다는 증거:**
```bash
docker inspect -f '{{.State.Pid}}' a      # → 12345
sudo readlink /proc/12345/ns/net          # net:[4026532500]
sudo readlink /proc/12400/ns/net          # net:[4026532600]  ← 다른 ID
readlink /proc/self/ns/net                # net:[4026531992]  ← 호스트, 또 다름
```

**8080은 왜 겹치면 안 되나?** 둘 다 **호스트 namespace 소속**이라서.
```bash
docker run -d -p 8080:80 nginx    # 성공
docker run -d -p 8080:80 nginx    # Error: port is already allocated
```

**8080 → A의 80 은 어떻게 도달하나? iptables DNAT.**
```
브라우저 → 192.168.0.10:8080 (호스트 namespace)
   ↓ iptables 규칙이 낚아챔 (docker가 자동으로 심음)
   ↓ 목적지를 172.17.0.2:80 으로 DNAT
   ↓ veth 랜선 → 컨테이너 A의 namespace
nginx가 자기 80포트에서 수신
```
```bash
sudo iptables -t nat -L DOCKER -n
# DNAT tcp dpt:8080 to:172.17.0.2:80
```

> **`-p`의 정체** = 포트를 "여는" 게 아니라, **서로 다른 두 namespace 사이에 iptables DNAT로 다리를 놓는 것.**
> `-p` 없이도 컨테이너끼리는 같은 브리지에 꽂혀 있어 IP로 직접 통신 가능하다.

#### ★★ 이것이 Pod를 이해하는 열쇠

**Pod = NET namespace를 공유하는 컨테이너 묶음**

```
Pod (NET namespace 1개)
├─ 컨테이너 A (Spring Boot) : 8080  ┐ 같은 통!
└─ 컨테이너 B (로그수집기)  : 9090  ┘
```

- IP가 하나 → 그 namespace의 IP
- **포트 공간도 하나** → 같은 Pod 안에서 8080을 둘이 쓰면 **충돌난다**
- 서로를 `localhost`로 호출 가능

Docker로 재현 가능:
```bash
docker run -d --name first nginx
docker run -d --name second --network=container:first alpine sleep 300
docker exec second ip addr                  # first와 똑같은 IP
docker exec second wget -qO- localhost:80   # first의 nginx 응답!
```
> k8s는 **pause 컨테이너**라는 껍데기를 먼저 띄워 namespace를 만들고, 나머지 컨테이너를 거기에 합류시킨다.

#### MNT namespace와 overlay의 관계

```bash
docker run --rm alpine ls /
# bin dev etc home lib ...  ← 호스트의 / 가 아님
```

**MNT namespace로 시야를 자르고, 그 안에 overlay로 만든 가짜 루트를 넣어주는 것.** 이것이 컨테이너 파일시스템의 정체다. (2장의 `overlay` 모듈이 여기서 쓰인다)

---

### 0-3. cgroup — 자원 격리 (성격이 다름)

| | namespace | cgroup |
|---|---|---|
| 격리하는 것 | **볼 수 있는 것** | **쓸 수 있는 양** |
| 위반하면 | 애초에 안 보여서 불가능 | 제한선에서 막힘 |
| 비유 | 칸막이 | 예산 배정 |

```bash
docker run --memory=512m --cpus=1.5 my-app
```
- 메모리 초과 → **OOMKilled** (커널이 죽임)
- CPU 초과 → **throttling** (느려짐, 죽진 않음)

> **차이의 이유**: 메모리는 이미 쓴 걸 회수할 수 없지만, CPU는 기다리게 하면 된다.

> **⚠️ 6장으로 이어지는 함정**: cgroup은 namespace가 **아니라서 `/proc/meminfo`를 가려주지 않는다.** 컨테이너 안에서 `free`를 치면 호스트 전체 메모리가 나온다. → JVM OOMKilled의 원인. **"제한"과 "시야 차단"이 별개**임을 보여주는 대표 사례.

---

### 0-4. VM 격리 — 하드웨어가 강제한다

컨테이너가 커널의 **소프트웨어 속임수**라면, VM은 **CPU가 물리적으로 차단**한다.

#### 부품 ① CPU 특권 모드 (Intel VT-x / AMD-V)

원래 CPU 권한 계층:
```
Ring 0 = 커널 (하드웨어 직접 조작 가능)
Ring 3 = 일반 앱
```

문제: 게스트 커널은 자기가 Ring 0인 줄 안다. 진짜 Ring 0를 주면 호스트를 장악한다.

해결: CPU에 **모드를 하나 더 추가**했다.
```
[VMX Root 모드]     ← 하이퍼바이저 (진짜 관리자)
[VMX Non-root 모드] ← 게스트 커널
     └─ 이 안에도 Ring 0~3이 다 있음 (게스트는 자기가 Ring 0라고 믿음)
```

게스트가 위험한 명령을 실행하면 **CPU가 하드웨어 레벨에서 낚아채** 하이퍼바이저로 넘긴다 (**VM Exit**).

> **비유**: 컨테이너는 "이 방 밖으로 나가지 마"라고 **말로 약속**한 것. VM은 **방문을 물리적으로 잠근** 것. 손잡이를 돌리는 순간 CPU가 감지해 경비원을 부른다.

#### 부품 ② 메모리 격리 (EPT / NPT)

```
게스트가 보는 주소  →  [EPT 하드웨어 변환표]  →  실제 물리 주소
    0x1000                                        0x8A3F1000
```

**CPU 안에 박힌 변환표가 강제로 주소를 바꾼다.** 게스트가 아무리 이상한 주소를 찔러도 자기 영역 밖으론 **물리적으로 접근 불가능**하다. 컨테이너엔 이런 게 없다 — 같은 커널의 같은 메모리 관리자를 쓰기 때문.

#### 커널은 몇 개인가

```
VM 3개  → 게스트 커널 3개 + 호스트 커널 1개 = 4개
컨테이너 3개 → 커널 1개 (다 같이 씀)
```

VM 안의 커널은 **진짜 완전한 커널**이다. 부팅도 하고 스케줄링도 하고 자기만의 namespace/cgroup도 있다. **그래서 VM 안에서 Docker가 돌아간다.**

#### 최종 비교

| | 컨테이너 | VM |
|---|---|---|
| 커널 | 1개 공유 | **VM마다 1개씩** |
| 격리 담당 | 커널 소프트웨어 (namespace) | **CPU 하드웨어** (VT-x, EPT) |
| 격리 대상 | 프로세스 목록·네트워크·파일시스템 | **CPU 명령어·메모리 주소·장치** |
| 부팅 | 없음 (프로세스 실행) | **진짜 부팅** (BIOS→커널→init) |
| 뚫으려면 | 커널 취약점 하나 | 하이퍼바이저 + CPU 취약점 |
| 무게 | MB, 즉시 | GB, 수십 초 |

> **비유**: 컨테이너는 **큰 사무실에 칸막이를 쳐 5개 회사가 입주**한 것. 각자 "우리 사무실"이라 여기지만 **불이 나면(커널 패닉) 다 같이 죽는다.** VM은 아예 **다른 건물**이다.

#### 격리 확인 실습

```bash
docker run --rm alpine ps aux        # PID: 자기가 1번
docker run --rm alpine hostname      # UTS: 다른 hostname
docker run --rm alpine ip addr       # NET: 다른 IP
docker run --rm alpine ls /          # MNT: 다른 루트

# ★ 격리를 "끄면" 어떻게 되는지 — 가장 인상적
docker run --rm --pid=host alpine ps aux | head -20
# → 호스트의 모든 프로세스가 보임! PID namespace를 안 만든 것

# namespace 실체 확인
docker run -d --name test alpine sleep 300
PID=$(docker inspect -f '{{.State.Pid}}' test)
sudo ls -l /proc/$PID/ns/     # pid, net, mnt, uts, ipc ... 각 ID
ls -l /proc/self/ns/          # 내 셸의 것과 비교 → ID가 다름
docker rm -f test
```

---

## 1장. 쉘 문법 — heredoc과 tee

2장 명령어를 읽기 위한 사전 문법.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

한 덩어리로 보이지만 3개 부품의 조립이다.

### 부품 ① `cat <<EOF ... EOF` (heredoc)

"지금부터 `EOF`가 나올 때까지 타이핑하는 걸 **한 덩어리 텍스트**로 취급"

```bash
cat <<EOF     ← "EOF 나올 때까지 받아적어"
overlay       ← 내용
br_netfilter  ← 내용
EOF           ← 끝 신호
```

`EOF`는 약속된 이름일 뿐. `cat <<끝` ... `끝` 도 동작한다.

### 부품 ② `|` (파이프)

왼쪽 명령의 **출력** → 오른쪽 명령의 **입력**

### 부품 ③ `sudo tee 파일경로`

받은 내용을 **파일에 저장하면서 화면에도 출력**. (T자 배관처럼 갈라져서 tee)

#### 왜 `sudo echo "..." > 파일` 이 아니라 tee인가? ★

```bash
sudo echo "overlay" > /etc/modules-load.d/k8s.conf
# → Permission denied!
```

**`>` (리다이렉션)은 sudo가 아니라 지금 내 셸이 처리하기 때문.**

```
sudo echo "overlay"  >  /etc/...
└─ 관리자 권한 ─┘   └─ 내 권한(일반 유저) ─┘
```

파일 만드는 담당자(셸)가 일반 유저라 튕긴다.
`tee`는 **자기가 직접 파일을 쓰는 프로그램**이므로 `sudo tee`면 관리자 권한으로 쓸 수 있다.

---

## 2장. k8s 설치 전 커널 설정 4단계

### 전체 명령

```bash
# 1. 부팅 시 자동 로드할 커널 모듈 설정 파일 생성
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 2. 커널 모듈 즉시 로드
sudo modprobe overlay
sudo modprobe br_netfilter

# 3. iptables가 브리지 트래픽을 처리하도록 sysctl 설정 추가
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 4. 재부팅 없이 sysctl 설정 즉시 적용
sudo sysctl --system
```

### 사전 개념 2개

| 개념 | 설명 | 비유 |
|---|---|---|
| **커널 모듈** | 필요할 때 꺼내 장착하는 커널 기능 조각 | 창고에서 꺼내 쓰는 연장 |
| **sysctl** | 커널 동작을 켜고 끄는 스위치판 (`/proc/sys/`) | 관리인 사무실 스위치 |

> **Java 비유**: 클래스 파일은 디스크에 있지만 쓰려면 클래스 로더가 메모리에 올려야 한다. `modprobe` = 클래스 로딩.

### ★ 핵심 패턴: 영구 + 즉시 (2단계 세트)

| 명령 | 지금 효과 | 재부팅 후 |
|---|---|---|
| 1번 (파일 생성) | 아무 일 없음 | 자동 로드됨 ✅ |
| 2번 (`modprobe`) | 즉시 로드 ✅ | 날아감 ❌ |
| 3번 (파일 생성) | 아무 일 없음 | 자동 적용됨 ✅ |
| 4번 (`sysctl --system`) | 즉시 적용 ✅ | — |

> **리눅스 설정은 항상 "파일에 적기(영구) + 지금 실행하기(즉시)" 2단계 세트다.**
> 하나만 하면 지금만 되거나 나중만 된다. 이 패턴은 계속 반복되니 기억할 것.

### 1번 — 부팅 시 로드할 모듈 목록

`/etc/modules-load.d/` 폴더는 `systemd-modules-load.service`가 부팅 때마다 읽는다.

> 비유: **정문 경비실에 붙인 "매일 아침 챙길 연장 목록"**

#### `overlay` — 컨테이너 이미지를 겹쳐 쌓는 파일시스템

OverlayFS. 여러 폴더를 **투명 필름처럼 겹쳐** 하나로 보이게 한다.

```
[레이어 3] 내 앱 jar        ┐
[레이어 2] JDK 설치         │ 겹치면 → 하나의 파일시스템처럼 보임
[레이어 1] Ubuntu 베이스    ┘
```

Docker 이미지가 레이어로 쌓이는 원리. 컨테이너 런타임(containerd)이 기본으로 쓴다.

#### `br_netfilter` — 브리지 패킷도 방화벽에 태우는 모듈

- **브리지(bridge)**: 소프트웨어 가상 스위치. 컨테이너들이 여기 랜선 꽂듯 연결됨
- **iptables/netfilter**: 리눅스 패킷 검문소

**문제**: 기본 상태에선 브리지 안에서 오가는 패킷을 iptables가 못 본다.

> 비유: 아파트 정문에 경비실(iptables)이 있는데, **같은 동 주민끼리 복도에서 주고받는 건 경비실을 안 거친다.**

**왜 문제인가**: k8s는 **kube-proxy가 iptables 규칙으로 Service 로드밸런싱**을 한다. Pod끼리 브리지 통신할 때 iptables를 안 거치면 **서비스 라우팅이 통째로 깨진다.** (7장에서 상세)

### 2번 — 지금 즉시 장착

```bash
sudo modprobe overlay        # module probe
sudo modprobe br_netfilter
```

확인:
```bash
lsmod | grep -E 'overlay|br_netfilter'
```

### 3번 — 커널 스위치 3개

`/etc/sysctl.d/` 도 부팅 시 자동으로 읽힌다. `= 1` 켜기, `= 0` 끄기.

#### ①② `bridge-nf-call-iptables` / `-ip6tables`

`br_netfilter` **모듈을 장착했지만 아직 전원 버튼을 안 누른 상태**. 이 스위치를 켜야 실제로 "브리지 패킷도 iptables 태우기"가 동작한다.

> ⚠️ **순서가 의미 있다**: 이 sysctl 키는 `br_netfilter` 모듈이 로드돼야 **존재 자체가 생긴다.**
> 모듈 없이 적용하면 `net.bridge.bridge-nf-call-iptables is an unknown key` 에러.
> **그래서 2번(modprobe)이 3번보다 먼저 온다.**

#### ③ `net.ipv4.ip_forward`

"나한테 온 게 아닌 패킷도 다른 데로 전달할까?" 스위치. 기본값 0.

일반 PC는 0이 맞다. 하지만 **k8s 노드는 라우터 역할을 해야 한다.**

```
Pod A (10.244.1.5) → 노드 → 다른 노드의 Pod B (10.244.2.7)
                      ↑ 내 앞으로 온 게 아닌데 전달해야 함
```

Pod IP는 노드 IP와 대역이 다르다. `ip_forward = 0`이면 노드가 "내 거 아니네" 하고 버려서 **클러스터 네트워크가 통째로 안 돈다.**

> 비유: **아파트 관리사무소를 우체국 중계소로 승격**시키는 스위치

### 4번 — 즉시 적용

```bash
sudo sysctl --system
```

읽는 순서:
```
/usr/lib/sysctl.d/*.conf    (배포판 기본값)
/run/sysctl.d/*.conf        (런타임)
/etc/sysctl.d/*.conf        (내가 만든 거 ← 여기)
/etc/sysctl.conf            (전통적 위치)
```

확인:
```bash
sysctl net.ipv4.ip_forward   # → net.ipv4.ip_forward = 1
```

### 요약표

| # | 명령 | 하는 일 | 시점 |
|---|---|---|---|
| 1 | 파일 → `/etc/modules-load.d/` | 부팅 시 로드할 **모듈 목록** 등록 | 다음 부팅부터 |
| 2 | `modprobe` | 그 모듈을 **지금 즉시** 장착 | 지금 |
| 3 | 파일 → `/etc/sysctl.d/` | 커널 **스위치 설정** 등록 | 다음 부팅부터 |
| 4 | `sysctl --system` | 그 스위치를 **지금 즉시** 켬 | 지금 |

**한 문장**: 컨테이너 이미지를 겹쳐 쌓을 도구(overlay)와 브리지 패킷을 방화벽에 태울 도구(br_netfilter)를 장착하고, 그 기능을 켜는 스위치 3개를 올려서 **k8s 네트워크가 돌아갈 바닥을 까는 작업.**

---

## 3장. k8s 계층 구조 — 컨테이너 / Pod / 노드 / 클러스터

### 계층 그림

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

### ⚠️ 흔한 오해

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

### 왜 Pod라는 층이 굳이 있나?

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

### ★ 계층별 구분 수단

| 계층 | 구분 수단 | 예시 | 유일성 범위 |
|---|---|---|---|
| 컨테이너 | **포트** | `:8080`, `:9090` | Pod 안에서 |
| Pod | **Pod IP** | `10.244.1.5` | **클러스터 전체** |
| 노드 | **노드 이름** (+ 노드 IP) | `node-01` / `192.168.0.11` | 클러스터 전체 |
| 클러스터 | **API 서버 주소** | `https://192.168.65.3:6443` | 전 세계 |

> **주의**: Pod IP는 노드 안이 아니라 **클러스터 전체에서 유일**하다. 노드가 달라도 절대 안 겹친다.

### 노드의 진짜 식별자는 IP가 아니라 "이름"

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

### 3-1. ⚠️ 컨트롤 플레인은 계층이 아니라 "역할"이다

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

계층은 **클러스터 → 노드 → Pod → 컨테이너 4단계**이고, 컨트롤 플레인은 이 계층에 들어가지 않는다. (자세한 내용은 10장)

---

### 3-2. Pod 안에는 무엇이 있나

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

#### ★ 공유되는 것 / 안 되는 것

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

### 3-3. pause 컨테이너 — 아무 일도 안 하는 주인공

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

#### 왜 필요한가 — pause 없으면 생기는 문제 4가지

| # | 문제 | 설명 |
|---|---|---|
| ① | **IP 변경** | 앱이 namespace 주인이면 앱 크래시 → refcount 0 → namespace 소멸 → 재시작 시 새 IP |
| ② | **주인 선정 불가** | 컨테이너 2개 이상일 때 누가 namespace를 만들지 정할 수 없음. 시작 순서가 곧 소유권이 됨 |
| ③ | **좀비 수거** | `shareProcessNamespace: true`일 때 PID 1이 고아 프로세스를 수거해야 함 |
| ④ | **준비 시점 불명확** | init 컨테이너도 네트워크가 필요한데, init이 namespace를 만들면 init 종료 시 사라짐 |

> **참조 카운트로 설명하면**: namespace는 가리키는 프로세스가 0개가 되면 소멸하는 커널 객체다(Java GC와 동일). pause가 **refcount를 1 이상으로 유지**해서 앱이 재시작해도 IP가 살아있는 것이다.

> **엄밀히는 필수가 아니다**: `ip netns add`처럼 bind mount로 NET namespace만 붙잡는 것도 가능하다. 다만 **PID namespace는 프로세스가 반드시 필요**하고, 어차피 프로세스를 띄울 거면 namespace 전체를 맡기는 게 깔끔하다.

#### pause는 PID 1인가? — 기본값에선 아니다

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

#### Pod 레벨 cgroup의 기준점

```
/sys/fs/cgroup/kubepods.slice/
  └── kubepods-...-pod<UID>.slice/          ← ★ Pod 레벨
      ├── memory.max = 640Mi                 (컨테이너 limits 합계 = 천장)
      ├── cri-containerd-<pause>.scope/
      ├── cri-containerd-<nginx>.scope/      memory.max = 512Mi
      └── cri-containerd-<sidecar>.scope/    memory.max = 128Mi
```

pause가 이 cgroup에 들어가서 **최소 1개 프로세스를 유지**한다(빈 cgroup은 정리될 수 있음). 그리고 Pod 레벨 한도가 **부모 천장**으로 작용한다 — 6장의 호스트↔VM 중첩과 같은 구조.

---

### 3-4. init 컨테이너 — 레이어가 아니라 순서

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

#### nginx 이미지의 원래 파일은?

nginx 이미지엔 원래 `/usr/share/nginx/html/index.html`이 있다. 그 경로에 볼륨을 마운트하면 **원래 내용은 가려진다** — 삭제가 아니라 **마운트가 경로를 덮은 것**이다. (책상 위 서류에 상자를 올려놓은 것과 같다)

#### ⚠️ `containers`는 필수 필드다

```yaml
spec:
  template:
    spec: {}     # ← error: spec.template.spec.containers: Required value
```

**컨테이너 없는 Pod는 만들어질 수 없다.** Pod는 그릇 자체가 아니라 **컨테이너들이 공유 환경을 갖게 하는 장치**이므로, 공유할 주체가 없으면 성립하지 않는다.

#### init vs 사이드카

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

## 4장. 네트워크 — 오버레이와 언더레이

### 주소 평면이 2개다

```
┌─ Pod 네트워크 (오버레이) ── 10.244.0.0/16 ─────────┐
│   Pod A: 10.244.1.5    Pod B: 10.244.2.7          │
└───────────────────────────────────────────────────┘
                        ↕ (얹혀 있음)
┌─ 노드 네트워크 (언더레이) ── 192.168.0.0/24 ───────┐
│   node-01: 192.168.0.11   node-02: 192.168.0.12   │
└───────────────────────────────────────────────────┘
```

| | 언더레이 (노드 네트워크) | 오버레이 (Pod 네트워크) |
|---|---|---|
| 정체 | 진짜 물리 랜선 / VPC 서브넷 | 소프트웨어로 그린 **가상 주소 체계** |
| 물리 장비가 아는가 | 안다 | **모른다** |

### 캡슐화 — 노드 IP로 한 번 더 포장

VXLAN 방식 기준:

```
Pod A(10.244.1.5) → Pod B(10.244.2.7) 로 보낼 때

[겉봉투: 192.168.0.11 → 192.168.0.12]  ← 물리 네트워크가 읽는 주소
  └ [속봉투: 10.244.1.5 → 10.244.2.7]  ← 도착해서 뜯으면 나오는 원래 패킷
```

물리 네트워크는 겉봉투만 보고 배달하고, 받은 노드가 겉봉투를 벗겨 속봉투를 Pod에 전달한다.

> 비유: 회사 내부 사번으로 편지를 보내는데 우체국은 사번을 모른다. 그래서 **회사 주소가 적힌 큰 봉투에 넣어** 보낸다. 도착하면 총무팀이 뜯어서 사번 보고 전달.

> **★ 여기서 2장과 연결**: 노드가 겉봉투를 뜯고 **자기 것도 아닌 속봉투를 Pod로 넘겨줘야** 하므로, `ip_forward = 1`이 켜져 있어야 한다.

### k8s 네트워크 모델 3대 전제

1. **모든 Pod는 고유한 IP를 가진다**
2. **모든 Pod는 NAT 없이 서로 직접 통신할 수 있어야 한다** (노드가 달라도)
3. 이걸 실제로 구현하는 게 **CNI 플러그인** (Calico, Flannel 등)

### 컨테이너 네트워크 구성 부품

| 개념 | 한 줄 설명 |
|---|---|
| **패킷** | 데이터 조각. 겉에 출발지·목적지 IP가 적힌 봉투 |
| **NIC** | 랜카드. 패킷이 드나드는 물리적 구멍 |
| **라우팅** | "이 목적지로 가려면 어느 길로?" 판단 |
| **브리지** | 소프트웨어로 만든 **가상 스위치** |
| **veth pair** | 양쪽 끝이 뚫린 가상 랜선. 한쪽은 컨테이너 안, 한쪽은 브리지에 |
| **NAT** | 봉투에 적힌 주소를 바꿔치기 |

```
[컨테이너 A] ──veth── ┐
                      ├─ [가상 브리지] ── [호스트 NIC] ── 외부
[컨테이너 B] ──veth── ┘
```

### iptables / netfilter

리눅스 커널 안의 **패킷 검문소**. 세 가지 일을 한다.

1. **필터링** — 통과 / 차단
2. **NAT** — 주소 바꿔치기
3. **리다이렉트** — 다른 곳으로 보내기

k8s는 이걸 **로드밸런싱 용도로 훔쳐 쓴다.** (7장)

---

### 4-1. netfilter와 iptables의 관계

> **⚠️ iptables는 IP 주소를 담는 테이블이 아니다.** IP를 **"조작하는 규칙"** 을 담는 곳이다.
> `ip` + `tables` = "IP 패킷을 다루는 규칙 표"

**netfilter = 커널에 박힌 패킷 처리 프레임워크.** 패킷 경로에 훅 5개가 박혀 있다.

```
          [랜카드로 패킷 도착]
                 │
          ① PREROUTING     ← 라우팅 판단 전
                 │
          ┌──라우팅 판단──┐
    (내 것)              (남의 것)
          │               │
      ② INPUT        ③ FORWARD   ← ip_forward=1이 여기를 켠다
          │               │
      [내 프로세스]        │
          │               │
      ④ OUTPUT            │
          └──⑤ POSTROUTING─┘  ← 나가기 직전
                 │
           [랜카드로 전송]
```

**패킷은 이 5개 지점을 반드시 지난다.** 각 지점에서 커널이 등록된 규칙을 확인하고 실행한다.

> **Spring 비유**: netfilter는 **필터 체인/인터셉터**다. 요청이 컨트롤러에 닿기 전 반드시 거치는 지점이 정해져 있고, 거기에 필터를 등록하는 것.

```
iptables (명령어, 사용자 공간)
    ↓ 규칙 등록
netfilter (커널 안의 실행 엔진)
    ↓ 실제 패킷 처리
```

규칙의 형태는 **"조건 → 동작"** 이다.
```bash
sudo iptables -t nat -L KUBE-SERVICES -n
# DNAT  tcp  dpt:80  to:10.244.1.5:8080
#       └조건┘        └────동작────┘
```

**4개 테이블** (이게 "tables"의 의미 — 용도별 규칙 묶음):

| 테이블 | 용도 |
|---|---|
| **filter** | 통과/차단 (방화벽) |
| **nat** | 주소 바꿔치기 ← **k8s Service가 쓰는 곳** |
| **mangle** | 패킷 헤더 수정 |
| **raw** | 연결 추적 제외 |

---

### 4-2. ★ 테이블 3형제 — 라우팅 / iptables / 소켓

| | **라우팅 테이블** | **iptables** | **소켓 테이블** |
|---|---|---|---|
| 질문 | "어느 **방향**으로?" | "**조작**할 게 있나?" | "**누구**에게 줄까?" |
| 계층 | L3 | L3/L4 | L4 |
| 내용 | 목적지 대역 → 출구 | 조건 → 동작 | 포트 → 프로세스 |
| 패킷 변조 | ❌ | ⭕ | ❌ |
| 명령어 | `ip route` | `iptables -L` | `ss -tlnp` |
| 비유 | 네비게이션 | 검문소 | 아파트 우편함 |

```bash
ip route
# default via 192.168.0.1 dev eth0     ← 모르는 곳은 게이트웨이로
# 10.244.0.0/16 dev cni0               ← Pod 대역은 이 인터페이스로

sudo iptables -t nat -L -n
# DNAT tcp dpt:8080 to:172.17.0.2:80   ← 주소를 바꿈

ss -tlnp
# LISTEN 0.0.0.0:80  users:(("nginx",pid=1234))   ← 프로세스 연결
```

#### 패킷 하나가 지나가는 순서

```
[패킷 도착]
   ↓
① netfilter PREROUTING → iptables nat 확인
   "10.96.0.42:80? → 10.244.2.7:8080으로 바꿔"     ★ Service 동작
   ↓
② 라우팅 테이블 조회
   "10.244.2.7은 cni0으로 가야겠네"
   ↓
③ netfilter INPUT → iptables filter 확인
   "차단 규칙 없네, 통과"
   ↓
④ 소켓 테이블 조회
   "8080? nginx 프로세스에게"
   ↓
[nginx가 받음]
```

**세 테이블이 순서대로 각자 다른 판단을 한다.** "Service = iptables"라는 말은, Service의 실체가 프로세스가 아니라 **DNAT 규칙 몇 줄**이라는 뜻이다.

---

### 4-3. ★ L4 vs L7 라우팅

#### L = Layer. 통신은 층으로 나뉜다

| 층 | 이름 | 다루는 것 | 예시 |
|---|---|---|---|
| **L7** | 응용 | **내용** | HTTP, DNS, gRPC |
| L4 | 전송 | **포트** | TCP, UDP |
| L3 | 네트워크 | **IP 주소** | IP, 라우터 |
| L2 | 데이터링크 | MAC 주소 | 이더넷, 스위치 |

각 층이 자기 정보를 봉투에 적어 아래로 넘긴다 — **봉투 안의 봉투**.

```
L7: "GET /users HTTP/1.1"      ← 편지 내용
L4: [포트 8080] + 위의 것       ← 겉봉에 "8080호"
L3: [IP 10.0.0.5] + 위의 것     ← "서울시 강남구..."
L2: [MAC 주소] + 위의 것         ← 배달 트럭 번호
```

#### 결정적 차이 — 봉투를 몇 개 뜯느냐

```
L4 라우팅: 겉봉투(IP + 포트)만 보고 배달. 안은 안 뜯음.
L7 라우팅: 편지를 꺼내 읽어보고 배달.
```

```
┌─ L3 헤더 ──────────────┐
│ 목적 IP: 10.96.0.42     │  ← L4가 보는 곳
├─ L4 헤더 ──────────────┤
│ 목적 포트: 80           │  ← 여기까지만
├─ L7 데이터 ────────────┤
│ GET /api/users HTTP/1.1 │
│ Host: api.example.com   │  ← L7은 여기까지 읽음
│ Cookie: session=abc123  │
└────────────────────────┘
```

**L4가 못 하는 것 4가지:**

| 못 하는 것 | 이유 |
|---|---|
| URL 경로 분기 (`/api` vs `/admin`) | URL이 봉투 안에 있음 |
| 도메인 분기 (`api.` vs `shop.`) | `Host:` 헤더가 L7 |
| HTTPS 처리 | 암호를 풀려면 내용을 봐야 함 |
| **요청 단위 분배** | 연결이 맺어질 때 한 번만 Pod 선택 |

마지막 항목이 실무에서 중요하다.
```
L4: 연결 1개 → Pod A 고정 → Keep-Alive 요청 100개 전부 Pod A ❌
L7: 요청마다 → A, B, C, A, B... ✅
```
**gRPC나 HTTP/2는 연결을 오래 유지하므로 L4 로드밸런싱이 사실상 무력해진다.**

#### 그럼 L4는 왜 쓰나

| | L4 | L7 |
|---|---|---|
| 속도 | **매우 빠름** | 느림 |
| 지원 프로토콜 | **TCP/UDP 전부** (MySQL, Redis, 게임) | HTTP 계열만 |
| 암호화 | 그대로 통과 (종단간 유지) | 풀었다 다시 암호화 |
| 리소스 | 적음 | 많음 |

**MySQL이나 Redis 앞에는 L7을 못 쓴다.** HTTP가 아니라 파싱할 게 없기 때문.

> **비유**: L4 = **우편 자동 분류기**(우편번호만 스캔, 초당 수천 통). L7 = **회사 안내데스크**(방문 목적을 듣고 부서 안내).

#### k8s에 대입

| | Service | Ingress |
|---|---|---|
| 계층 | **L4** | **L7** |
| 구현 | iptables (**커널**) | nginx Pod (사용자 공간) |
| 분배 기준 | 랜덤/라운드로빈 | Host, Path, 헤더 |
| 프로토콜 | TCP/UDP 전부 | HTTP/HTTPS |
| TLS | 못 함 | **종료 가능** |

**Service가 L4일 수밖에 없는 이유**: Service의 실체가 iptables이고, iptables는 커널의 netfilter다. **커널은 성능상 HTTP를 파싱하지 않는다.** 반면 Ingress Controller는 nginx라는 일반 프로그램이라 파싱이 가능하다.

```bash
# 증거 — iptables 규칙 어디에도 URL 조건이 없다
kubectl expose deployment web --port=80
CIP=$(kubectl get svc web -o jsonpath='{.spec.clusterIP}')
sudo iptables -t nat -L -n | grep $CIP
# → 목적지 IP:포트만. 경로/도메인 조건 없음 ★
```

#### 왜 L1·L5·L6 라우팅은 없나

**라우팅 = 고를 후보가 있는 층에서만 쓰는 말.**

| 층 | 후보 | 명칭 |
|---|---|---|
| L1 | ❌ 전선 하나 | 전송 |
| L2 | 🔺 표에서 조회(정답 하나) | **스위칭** |
| L3 | ⭕ 경로 선택 | **라우팅** (원조) |
| L4 | ⭕ 서버 선택 | L4 로드밸런싱 |
| L5·L6 | ❌ 독립 헤더 없음 | — |
| L7 | ⭕ 서비스 선택 | L7 라우팅 |

**L5·L6가 없는 진짜 이유: TCP/IP 모델에 그 층이 아예 없다.**

```
OSI 7계층 (이론)          TCP/IP 4계층 (현실)
L7 응용 / L6 표현 / L5 세션  →  응용 계층
L4 전송                      →  전송 계층
L3 네트워크                  →  인터넷 계층
L2 데이터링크 / L1 물리       →  링크 계층
```

> **`L2·L3·L4·L7`은 OSI 번호를 빌려 쓰는 관습어다.** TCP/IP 계층에는 번호가 없다. L1은 링크 계층에, L5·L6은 응용 계층에 **흡수**된 것이지 사라진 게 아니다 (세션 → TCP 연결/쿠키, 암호화 → TLS, 인코딩 → `Content-Encoding` 헤더).

**2·3·4·7만 언급되는 이유**: 그 층들만 **고유한 주소와 헤더**를 가져서 장비와 라우팅의 기준이 될 수 있다.

| 층 | 고유 주소 | 대표 도구 |
|---|---|---|
| L2 | MAC | 스위치, 브리지 ← **docker0** |
| L3 | IP | 라우터 ← **ip route** |
| L4 | 포트 | 방화벽, NLB ← **iptables, k8s Service** |
| L7 | URL/Host | 프록시, ALB ← **nginx, k8s Ingress** |

> **TLS의 애매함**: TLS는 TCP 위, HTTP 아래에 끼어 있어 OSI로 깔끔히 매핑되지 않는다. 그래서 실무에선 계층 번호 대신 **"TLS 종료(termination)"** vs **"패스스루(passthrough)"** 로 부른다. Ingress가 TLS를 종료할 수 있는 건 암호를 풀어야 Host/Path를 읽기 때문이고, Service(L4)는 암호화된 바이트 덩어리를 그대로 통과시킬 수밖에 없다.

---

## 5장. kubectl context와 kubeconfig

### 실제 겪은 상황

```
$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop   docker-desktop   docker-desktop
└─ 비어있음!

$ kubectl config use-context docker-desktop
Switched to context "docker-desktop".

$ kubectl config current-context
docker-desktop
```

`CURRENT`가 비어 있으면 `kubectl get pods` 시 이런 에러가 난다:
```
The connection to the server localhost:8080 was refused
```
→ **"어느 클러스터한테 말 걸어야 할지 모르겠다"** 는 뜻. (`localhost:8080`은 설정을 못 찾았을 때 쓰는 하드코딩 기본값)

### kubectl은 사실 아무것도 모른다

**`kubectl`은 클러스터가 아니다. 그냥 HTTP 클라이언트다.**

```
[kubectl] --HTTP 요청--> [API Server] --> [클러스터]
   ↑ 그냥 Postman 같은 놈
```

`kubectl get pods` 의 실체:
```
GET https://192.168.65.3:6443/api/v1/namespaces/default/pods
Authorization: (인증서 또는 토큰)
```

kubectl은 3가지를 **어디선가 알아내야** 한다:
1. **주소** — 어느 서버로? (`https://...:6443`)
2. **신분증** — 나는 누구? (인증서/토큰)
3. **작업 위치** — 어느 네임스페이스?

매번 손으로 치면 지옥이므로 **파일에 적어두고 이름표를 붙인 것**이 context다.

### kubeconfig 구조 (`~/.kube/config`)

```yaml
clusters:              # ① 어디로 (주소록)
- name: docker-desktop
  cluster:
    server: https://127.0.0.1:6443
    certificate-authority-data: LS0tLS1...

users:                 # ② 누구로 (신분증 보관함)
- name: docker-desktop
  user:
    client-certificate-data: LS0tLS1...
    client-key-data: LS0tLS1...

contexts:              # ③ ①+②+네임스페이스 조합에 이름 붙이기
- name: docker-desktop
  context:
    cluster: docker-desktop    # ①에서 고름
    user: docker-desktop       # ②에서 고름
    namespace: default

current-context: docker-desktop   # ← use-context가 바꾸는 게 이 한 줄!
```

### ★ 핵심 공식

```
context = 클러스터(어디로) + 유저(누구로) + 네임스페이스(어느 방에서)
```

`get-contexts` 출력 컬럼이 정확히 이것:

| CURRENT | NAME | CLUSTER | AUTHINFO | NAMESPACE |
|---|---|---|---|---|
| 지금 이거? | 조합 이름 | 어디로 | 누구로 | 어느 방 |

> 셋 다 이름이 `docker-desktop`인 건 **서로 다른 3개 항목이 우연히 이름이 같은 것**뿐. Docker Desktop이 다 같은 이름으로 만들어서 그렇다.

### 왜 전환이 필요한가

실무에선 이렇게 된다:

```
CURRENT   NAME              CLUSTER        NAMESPACE
          docker-desktop    docker-desktop
*         minikube          minikube       dev
          basecamp-prod     eks-prod       production
          basecamp-stage    eks-stage      staging
```

`kubectl delete deployment api-server` 를 쳤을 때:
- context가 `minikube` → 실습 환경 날아감. 괜찮음.
- context가 `basecamp-prod` → **운영 서버 내려감.**

**같은 명령어인데 context에 따라 결과가 완전히 달라진다.** context는 "이 명령이 어디에 떨어질지"를 결정하는 조준경.

> **★ Spring 비유 (가장 정확)**
> `application-dev.yml` / `application-prod.yml` 이 있고, `--spring.profiles.active=prod` 하나로 접속 DB가 바뀌는 것과 완전히 같다.
>
> | Spring | kubectl |
> |---|---|
> | `application.yml` 모음 | `~/.kube/config` |
> | profile (dev/prod) | context |
> | `--spring.profiles.active` | `use-context` |
>
> 잘못된 profile로 띄우면 운영 DB 건드리는 것도 똑같다.

### 명령어 정리

```bash
# 목록 (CURRENT의 * 가 현재)
kubectl config get-contexts

# 현재 것 한 줄로
kubectl config current-context

# 전환
kubectl config use-context docker-desktop

# 설정 전체 보기 (인증서는 REDACTED)
kubectl config view

# 현재 context의 기본 네임스페이스 변경 (자주 씀)
kubectl config set-context --current --namespace=my-app

# 전환 없이 한 번만 다른 context로
kubectl get pods --context=minikube
```

### 사고 방지 습관

```bash
# 1. 위험한 명령 전 확인
kubectl config current-context && kubectl delete deployment xxx

# 2. 프롬프트에 context 띄우기 (kube-ps1)
(⎈|basecamp-prod:production) user@DESKTOP:~$

# 3. 편의 도구
kubectx              # context 목록 + 선택
kubens my-app        # 네임스페이스 빠른 전환
alias k=kubectl      # 오타 고통 반감
```

---

## 6장. cgroup 중첩 구조와 JVM OOMKilled

### 질문: 호스트가 VM에 cgroup 걸고, VM이 컨테이너에 또 거는가?

**맞다. 단, "겹쳐서 걸린다"기보다 완전히 따로 노는 두 개의 시스템이다.**

cgroup은 **커널에 소속된 기능**이다. 커널이 2개면 cgroup 트리도 2개고, **서로의 존재를 모른다.**

```
┌─ 호스트 커널 (Windows Hyper-V / Linux) ─────────────┐
│                                                     │
│  cgroup 트리 #1  ← 호스트 커널이 관리                │
│   └─ "VM 프로세스야, CPU 4코어 / RAM 8GB까지만"      │
│         │                                           │
│         ▼                                           │
│  ┌─ VM (게스트 리눅스 커널) ──────────────────────┐  │
│  │  cgroup 트리 #2  ← 게스트 커널이 관리           │  │
│  │   ├─ 컨테이너 A: CPU 1코어 / RAM 512MB          │  │
│  │   └─ 컨테이너 B: CPU 0.5코어 / RAM 256MB        │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**트리 #1과 #2는 부모-자식이 아니다.** 서로 다른 커널이 각자 `/sys/fs/cgroup`에 만든 별개 트리.

### 게스트는 호스트의 cgroup을 볼 수 없다

```bash
# VM 안에서
$ free -h
              total
Mem:          7.7Gi     ← "내 컴퓨터 RAM이 8GB구나"
```

게스트 커널은 이걸 **cgroup 제한이 아니라 하드웨어 스펙**으로 인식한다. 하이퍼바이저가 가상 하드웨어로 만들어 보여준 것이기 때문.

> 비유: 회사가 팀에 예산 8천만원 배정 → 팀장이 팀원에게 "너 1천만원까지" 제한. 팀원은 **회사 전체 예산이 얼마인지, 왜 우리 팀이 8천만원인지 알 수 없다.**

### 두 제한은 곱해지지 않고, 작은 쪽이 이긴다

```
컨테이너: "나 16GB 쓸래"   ← 게스트 cgroup은 허락
VM 전체:  8GB뿐            ← 호스트가 막음
결과: 8GB 근처에서 OOM
```

게스트 커널은 16GB를 못 준다는 걸 **미리 알 수 없다.** 실제로 요청하다 물리적으로 없어서 터진다.
→ 실무에선 컨테이너 제한의 총합이 VM 크기를 넘지 않게 설계.

### 내 환경 (Docker Desktop on Windows)

```
Windows
 └─ Hyper-V가 WSL2 VM 생성
     ├─ 크기: %USERPROFILE%\.wslconfig    ← 호스트 제한
     │    [wsl2]
     │    memory=8GB
     │    processors=4
     │
     └─ WSL2 리눅스 커널
         └─ cgroup v2
             └─ docker run --memory=512m  ← 게스트 제한
```

> **VM 안에 컨테이너가 들어있는 중첩 구조.** 2장의 커널 설정도 Windows 커널이 아니라 **WSL2 리눅스 커널**에 적용된다.

### ★ 실전 payoff — JVM이 여기서 터진다

Java 8 초기 버전의 유명한 사고:

```
컨테이너 제한: 512MB
JVM이 본 것:   VM 전체 메모리 8GB
JVM 판단:      "힙 기본값 = 물리메모리의 1/4 → 2GB!"
결과:          힙 늘리다 512MB 초과 → OOMKilled → 파드 재시작 무한반복
```

**원인이 정확히 이 중첩 구조.** JVM이 `/proc/meminfo`를 읽었는데, 그건 **게스트 커널 전체 메모리**지 **자기 cgroup 제한**이 아니었다. 컨테이너는 프로세스일 뿐이라 `/proc`이 호스트(=VM) 것을 그대로 보여준다.

**해결**: `-XX:+UseContainerSupport` — JVM이 `/proc/meminfo` 대신 **cgroup 파일을 직접 읽도록** 변경. Java 8u191+, Java 10+ 부터 기본 활성화.

```bash
# 실무 권장
java -XX:MaxRAMPercentage=75.0 -jar app.jar
```

`-Xmx` 고정값보다 **비율**이 낫다. 컨테이너 메모리 제한을 바꿔도 자동으로 따라간다.

### 직접 확인

```bash
# WSL2 안 — 게스트가 인식하는 "하드웨어" 크기
free -h
nproc
ls /sys/fs/cgroup/

# 컨테이너 자기 제한
docker run --rm --memory=512m alpine cat /sys/fs/cgroup/memory.max
# → 536870912  (= 512MB)

# 같은 컨테이너에서 free
docker run --rm --memory=512m alpine free -m
# → VM 전체 메모리가 나옴! (제한값 아님)
```

> **마지막 두 개를 꼭 같이 쳐볼 것.** `memory.max`는 512MB인데 `free`는 8GB — 이 불일치가 "따로 노는 두 층"의 증거이자 JVM 사고의 원인.

---

## 7장. 서비스 디스커버리 — Service + CoreDNS

### 관통하는 원리

> **IP는 변한다. 이름은 안 변한다. 그래서 이름으로 부르고, 이름→IP 변환은 시스템이 알아서 한다.**

이것이 **간접 계층(indirection layer)**. 노드 이름 식별도, 서비스 디스커버리도 이 원리의 서로 다른 적용.

### 왜 IP로 직접 부르면 안 되나

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

### 부품 ① Service — 안 변하는 가짜 IP

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

> **★ 2장과 완전히 연결되는 지점**
> `bridge-nf-call-iptables = 1` 을 켠 이유가 정확히 이것.
> Pod가 브리지로 패킷을 보낼 때 iptables를 안 거치면 **이 낚아채기가 발동 자체를 안 한다.**
> Service IP로 보낸 패킷이 허공으로 사라진다.

Pod가 죽고 새로 뜨면? kube-proxy가 **iptables 규칙만 갱신**. Service IP는 그대로. 호출하는 쪽은 아무것도 몰라도 된다.

### 부품 ② CoreDNS — 이름을 IP로

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

### ★ 최종 흐름 — 간접 계층 2겹

```
"payment-service"
  → [CoreDNS]  → 10.96.0.42   (Service ClusterIP, 안 변함)
  → [iptables] → 10.244.2.7   (실제 Pod IP, 계속 변함)
```

**이름 → 안 변하는 가짜 IP → 계속 변하는 진짜 IP**

### 노드 이름 vs 서비스 디스커버리 — 뭐가 다른가

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

### Spring Cloud 대응표

| Spring Cloud | Kubernetes | 하는 일 |
|---|---|---|
| Eureka Server | **CoreDNS + etcd** | 서비스 목록 저장소 |
| Eureka Client 등록 | **자동** (Pod 뜨면 Endpoints 등록) | 나 여기 있다고 알림 |
| `@LoadBalanced RestTemplate` | **iptables (kube-proxy)** | 인스턴스 중 하나 선택 |
| Ribbon | **kube-proxy** | 로드밸런싱 |
| `http://payment-service/pay` | `http://payment-service/pay` | **문법이 똑같음!** |

**핵심 차이**: Spring Cloud는 **애플리케이션 레벨**(라이브러리)에서, k8s는 **인프라 레벨**(커널)에서 처리.

→ k8s를 쓰면 Eureka가 필요 없어진다. 언어도 안 가린다. Java든 Python이든 `http://payment-service` 로 부르면 끝. **라이브러리 의존성 0.**

### 예외 — Headless Service

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

## 8장. 워크로드 오브젝트 — Deployment / ReplicaSet / Service

### 왜 Pod만으로는 안 되나 — 사고 3종

Pod를 직접 하나 띄우면 이런 일이 생긴다.

| 사고 | 내용 | 해결책 |
|---|---|---|
| ① **부활 불가** | Pod가 죽으면 아무도 안 살려줌 | ReplicaSet |
| ② **수동 확장** | 3개로 늘리려면 손으로 3번 | ReplicaSet |
| ③ **배포 중단** | 1.0 지우고 2.0 띄우는 사이 서비스 끊김 | Deployment |
| ④ **IP 변경** | 배포하면 Pod IP가 전부 바뀜 | Service |

### 카페 비유

| k8s | 카페 | 하는 일 |
|---|---|---|
| **Pod** | 알바생 한 명 | 실제로 커피를 만드는 사람 |
| **ReplicaSet** | 인원 관리 규칙 | **"항상 3명"** 을 지킴 |
| **Deployment** | 사장님 | 알바를 **천천히 교체**, 문제 생기면 되돌림 |
| **Service** | 가게 대표 전화번호 | 손님은 이 번호만 알면 됨 |

---

### 8-1. ReplicaSet — 개수를 지키는 감시자

하는 일은 **딱 하나: 개수 세기.**

```yaml
replicas: 3
```

```
[감시] 지금 몇 개? → 3개. 아무것도 안 함.
[감시] 지금 몇 개? → 2개! (하나 죽음) → 즉시 1개 생성
[감시] 지금 몇 개? → 4개! (실수로 추가) → 즉시 1개 삭제
```

#### ★ 선언형(Declarative) — k8s 전체를 관통하는 사고방식

| | 명령형 (Imperative) | 선언형 (Declarative) |
|---|---|---|
| 말하는 법 | "Pod 하나 만들어" | "Pod가 3개인 **상태를 유지**해" |
| 누가 관리 | 내가 계속 봐야 함 | **시스템이 알아서** |
| 비유 | "에어컨 켜" | "온도 24도 유지해" |

**원하는 상태(desired state)만 적으면 k8s가 현재 상태를 거기에 맞춘다.** 이 루프를 계속 도는 게 **컨트롤러**.

> **실무 주의**: ReplicaSet은 직접 만들 일이 거의 없다. **Deployment가 대신 만들어준다.**

---

### 8-2. Deployment — 버전을 갈아끼우는 사장님

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

#### ★ 소유 관계 (가장 헷갈리는 부분)

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

#### 롤링 업데이트 — Deployment의 존재 이유

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

#### 롤백 — 옛날 ReplicaSet을 안 지우는 이유

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

### 8-3. Service — 안 변하는 대표 전화번호

#### 문제: 배포하면 Pod IP가 전부 바뀐다

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

#### ★ 라벨(label) — k8s에서 가장 중요한 연결 방식

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

#### Service 종류

| 종류 | 접근 범위 | 언제 |
|---|---|---|
| **ClusterIP** (기본) | 클러스터 **안**에서만 | 백엔드끼리 통신 |
| **NodePort** | 노드IP:포트로 외부에서 | 테스트용 |
| **LoadBalancer** | 클라우드 LB 경유 | 실서비스 (AWS면 ELB 자동 생성) |

---

### 8-4. 전체 그림

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

### 8-5. Spring 치환표

| k8s | Spring | 공통점 |
|---|---|---|
| Deployment | 배포 설정 | 어떻게 만들지 선언 |
| ReplicaSet | **커넥션 풀 (min/max)** | **개수를 유지**하는 관리자 |
| Pod | 실제 인스턴스 | 일하는 놈 |
| Service | `@Autowired` / DI 컨테이너 | **구현체를 몰라도 이름으로** 접근 |
| 라벨 셀렉터 | `@Qualifier` | 이름표로 대상 특정 |

> HikariCP가 죽은 커넥션을 알아서 채워 최소 개수를 유지하는 것 = ReplicaSet의 동작 원리와 동일.

---

### 8-6. ★ "관리한다"는 게 구체적으로 무엇인가

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

#### 시나리오별 동작

| 사건 | 루프의 반응 |
|---|---|
| 최초 생성 | 0개 → 3개 부족 → template으로 3개 생성 |
| Pod 삭제 | 2개 → 1개 부족 → 1개 생성 (수 초 내) |
| **노드 장애** | 정상 1개 → 2개 부족 → 다른 노드에 2개 생성 (**자가 치유**) |
| `scale --replicas=5` | 3개 → 2개 생성 |

**자가 치유는 별도 기능이 아니라 "세고 맞추기"의 부산물이다.**

#### ★★ 라벨을 떼면 — 이게 이해의 열쇠

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

#### template은 언제 쓰이나 — Pod를 새로 만들 때만

```bash
kubectl set image deployment/nginx-deploy nginx-container=nginx:1.25
```
**기존 Pod가 변신하는 게 아니다.** 새 template으로 새 Pod를 만들고 옛 Pod를 지운다.

> **Pod는 사실상 불변(immutable)이다.** 이미지도 환경변수도 수정되지 않는다. 바꾸려면 죽이고 새로 만드는 것뿐 — **불변 인프라(immutable infrastructure)** 원칙.
>
> **Java 비유**: template은 **생성자 인자**다. 객체 생성 시에만 쓰이고 기존 객체를 바꾸지 않는다. `String`이 불변이라 `replace()`가 새 객체를 반환하는 것과 같다.

#### Deployment는 Pod를 직접 안 건드린다

```
① Deployment: "template이 바뀌었네. 새 RS 만들자"
② RS(new) 생성, replicas=0
③ Deployment: RS(new)=1, RS(old)=2     ← 숫자만 조절
④ 각 RS가 자기 루프로 Pod 조절
⑤⑥ 반복 → RS(new)=3, RS(old)=0
```

**Deployment는 ReplicaSet의 숫자만 조절하고, 실제 Pod 생성/삭제는 각 RS가 자기 루프로 한다.**

---

### 8-7. pause vs Service — 둘 다 "IP 안정성"인데 뭐가 다른가

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

#### 안정성 계층 전체

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

## 9장. 오브젝트 구조 — metadata / spec / status, Namespace, Label

### 9-1. 모든 k8s 오브젝트는 3부 구성

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

#### ★ 왜 labels는 metadata, selector는 spec인가

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

### 9-2. Label — 붙이는 쪽과 찾는 쪽은 항상 짝이다

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

#### ⚠️ Deployment에 라벨이 세 군데 나오는 이유

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

#### 셀렉터 조건은 AND

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

### 9-3. Namespace vs Label — 폴더 vs 태그

| | Namespace | Label |
|---|---|---|
| **개수** | 오브젝트당 **1개** | **여러 개** |
| **변경** | ❌ **불가능** (immutable) | ✅ 언제든 |
| **역할** | **어디 소속인가** | **어떤 특성인가** |
| 이름 유일성 | Namespace 안에서 유일 | 무관 |
| RBAC/Quota | ⭕ 걸 수 있음 | ❌ |
| 삭제 시 | **안의 모든 것이 삭제** | 꼬리표만 제거 |

#### Namespace는 "주소의 일부"라서 하나뿐이다

```
etcd 경로:  /registry/pods/production/my-app
                          └ns┘      └name┘
```

파일이 두 폴더에 동시에 있을 수 없듯, Pod도 두 Namespace에 못 있는다. **그래서 변경도 불가능하다** — 지우고 새로 만들어야 한다.

#### name과 namespace의 차이

```
metadata.namespace  →  폴더
metadata.name       →  파일 이름
둘을 합쳐야         →  전체 주소
```

```bash
kubectl get pod my-app                # 실은 -n default 가 생략된 것
kubectl get pod my-app -n production  # 완전한 형태
```

#### Namespace가 필요한 이유 3가지

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

#### 함께 작동하는 방식

**Service의 selector는 같은 Namespace 안에서만 Label을 찾는다.**

> **SQL 비유**
> ```sql
> SELECT * FROM pods
> WHERE namespace = 'prod'         -- Namespace: 검색 범위
>   AND labels->>'app' = 'web'     -- Label: 검색 조건
> ```

---

### 9-4. ⚠️ k8s Namespace vs Linux namespace — 이름만 같다

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

#### 가장 중요한 오해: 네트워크가 격리되지 않는다

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

#### Namespaced vs Cluster-scoped 오브젝트

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

## 10장. 클러스터 아키텍처 — 누가 실제로 실행하는가

지금까지가 "무엇을 만드나"였다면, 이 장은 **"누가 그걸 만들어주나"** 다.

### 10-1. 큰 그림 — 두 덩어리

```
┌─ Control Plane (두뇌) ─────────────────────────────┐
│   [kube-apiserver] ← 유일한 관문                    │
│         ↕                                          │
│      [etcd]        ← 유일한 저장소                  │
│   [kube-scheduler]          [controller-manager]   │
│    어느 노드에 놓을까?        원하는 상태 맞추기      │
└────────────────────────────────────────────────────┘
                        ↕ (모든 통신은 API Server 경유)
┌─ Worker Node (근육) ───────────────────────────────┐
│   [kubelet]      노드 관리인, 컨테이너 실행 감독     │
│   [kube-proxy]   iptables 규칙 관리 (Service 구현)  │
│   [containerd]   실제 컨테이너 런타임                │
│         └─ Pod, Pod, Pod                           │
└────────────────────────────────────────────────────┘
```

> **주의**: 이 두 박스는 **서로 다른 노드**다 (3-1장). Control Plane도 결국 **노드 위에서 도는 Pod들**이다.

```bash
kubectl get pods -n kube-system -o wide
# kube-apiserver-docker-desktop   NODE: docker-desktop   ← 노드 위!
# etcd-docker-desktop
# kube-scheduler-docker-desktop
```

---

### 10-2. Control Plane 4대 컴포넌트

#### ① kube-apiserver — 유일한 관문

```
kubectl ────┐
kubelet ────┼──→ [API Server] ──→ etcd
scheduler ──┤
controller ─┘
```

> **★ 가장 중요한 설계 원칙: 컴포넌트끼리 직접 대화하지 않는다.**
> Scheduler가 kubelet에게 "이 Pod 띄워줘"라고 **직접 말하지 않는다.** API Server에 기록만 하고 kubelet이 읽어간다.

> **비유**: 다들 직접 연락하는 대신 **전부 사내 게시판에만 글을 올리는 것.** 추적이 되고, 한 명이 빠져도 다른 사람이 읽을 수 있고, 게시판만 지키면 보안이 된다.

하는 일: **인증 → 인가(RBAC) → Admission Control(검증/기본값 주입) → etcd 읽기/쓰기**

> kubeconfig의 `server: https://...:6443`이 바로 이 주소다 (5장). context 전환 = **어느 API Server에 말을 걸지** 바꾸기.

#### ② etcd — 유일한 저장소

```
/registry/pods/default/my-app-abc12       → Pod 정의
/registry/deployments/default/my-app      → Deployment 정의
```

| 특징 | 내용 |
|---|---|
| 백업 | **etcd만 백업하면 클러스터 구성 전체 복구 가능** |
| 접근 | **API Server만** 접근. kubelet도 scheduler도 직접 못 붙음 |
| watch | 값이 바뀌면 구독자에게 **밀어줌(push)** ← 실시간 반응의 비결 |

> **Spring 비유**: etcd = DB, API Server = 유일하게 DB에 붙는 서버. **DAO 계층을 하나로 강제한 구조.**

#### ③ kube-scheduler — 배치만 결정, 실행은 안 함

```
1. nodeName이 비어있는 Pod 발견
2. 필터링(Filtering): 못 놓는 노드 제외 (메모리 부족, 라벨 불일치, taint)
3. 점수매기기(Scoring): 자원 여유·이미지 보유 등으로 점수
4. 1등 노드 이름을 API Server에 기록 → 끝
```

**스케줄러는 컨테이너를 띄우지 않는다.** `nodeName: node-02`라고 **적기만** 한다.

> **비유**: 인사팀이 "김철수는 3층 영업부 배치"라고 **발령만 내는 것**. 자리 세팅은 3층 담당(kubelet)이 한다.

#### ④ kube-controller-manager — 컨트롤러 루프 뭉치

```
while (true) {
    현재상태 = API Server에서 읽기
    if (현재상태 != 원하는상태) 차이를 메우는 행동
}
```
이것이 **reconciliation loop(조정 루프)** — 선언형의 실체다.

| 컨트롤러 | 하는 일 |
|---|---|
| **Deployment** | ReplicaSet 생성, 롤링 업데이트 조율 |
| **ReplicaSet** | Pod 개수 유지 (8-6장의 루프) |
| **Node** | 하트비트 감시, 끊기면 Pod 퇴거 |
| **Endpoints** | 라벨 매칭해서 Service ↔ Pod IP 갱신 |
| **Job / CronJob** | 배치 작업 |

> **Spring 비유**: `@Scheduled(fixedDelay=1000)` 붙은 배치 잡 수십 개가 각자 담당 리소스를 감시하는 것.

---

### 10-3. Worker Node 3대 컴포넌트

#### ① kubelet — 노드의 관리인

```bash
systemctl status kubelet    # ★ 컨테이너가 아니라 진짜 systemd 서비스
```

1. API Server에 watch: "내 노드에 배정된 Pod 있어?"
2. CRI로 컨테이너 런타임에 실행 요청
3. Pod 상태를 API Server에 계속 보고 (하트비트)
4. **Probe 실행** (liveness/readiness)
5. 볼륨 마운트, ConfigMap/Secret 가져오기

**kubelet은 자기 노드 것만 신경 쓴다.**

#### ② kube-proxy — Service를 실제로 구현

```
Endpoints 변경 감지 (watch)
   ↓
각 노드의 iptables 규칙 재작성
   ↓
10.96.0.42:80 → 10.244.2.11:8080 (DNAT)
```

> **2장이 여기서 회수된다**: `bridge-nf-call-iptables=1`을 켠 이유가 이것. Pod가 브리지로 패킷을 보낼 때 iptables를 안 거치면 **kube-proxy가 심은 규칙이 발동 자체를 안 한다.**

#### ③ 컨테이너 런타임 (containerd / CRI-O)

```
kubelet ──[CRI]──> containerd ──> runc ──> 리눅스 커널 (namespace/cgroup)
```

**CRI(Container Runtime Interface)** = kubelet과 런타임 사이의 표준 인터페이스.

> **Java 비유**: CRI = JDBC. 인터페이스만 맞으면 드라이버를 갈아끼울 수 있다.

> k8s 1.24부터 dockershim이 제거됐지만, Docker가 내부적으로 containerd를 쓰고 이미지가 OCI 표준이라 **이미지 호환성은 그대로**다.

---

### 10-4. ★ `kubectl apply` 했을 때 실제로 벌어지는 일

```
① kubectl apply -f deployment.yaml
      ↓ HTTPS (kubeconfig의 인증서로)
② API Server: 인증 → 인가(RBAC) → Admission
      ↓
③ etcd에 Deployment 저장 ✅  ← kubectl은 여기서 "created" 출력하고 끝!
      ↓ watch 이벤트
④ Deployment 컨트롤러: "ReplicaSet이 없네?" → RS 생성 요청
      ↓ watch
⑤ ReplicaSet 컨트롤러: "Pod가 0개인데 3개여야 하네?" → Pod 3개 생성
         (이때 Pod의 nodeName은 비어있음!)
      ↓ watch
⑥ Scheduler: 필터링 + 점수매기기 → nodeName: node-02 기록
      ↓ watch
⑦ node-02의 kubelet: "내 노드에 배정됐네" → CRI로 containerd에 요청
      ↓
⑧ containerd: 이미지 pull → namespace 생성 → cgroup 설정 → 실행
      ↓
⑨ kubelet이 상태 보고 → API Server → etcd (status: Running)
      ↓ 동시에
⑩ Endpoints 컨트롤러가 라벨 매칭 → Endpoints 갱신
      ↓ watch
⑪ 각 노드의 kube-proxy가 iptables 갱신 → Service로 트래픽 도달 가능
```

**여기서 배울 3가지:**

1. **kubectl은 ③번에서 이미 끝난다.** `created`는 "etcd에 저장됨"이지 "Pod가 떴음"이 아니다. 그래서 바로 `get pods`를 치면 `ContainerCreating`이 보인다.
2. **모든 화살표가 API Server를 지난다.** 컨트롤러가 스케줄러에게 직접 말하는 화살표가 하나도 없다 — 전부 **etcd에 쓰고 watch로 감지**. 결합도가 0이라 스케줄러가 죽어도 나머지는 돈다.
3. **각 단계가 독립적으로 재시도된다.** 어느 단계가 실패해도 루프가 다시 돌면서 복구를 시도한다.

---

### 10-5. 닭과 달걀 — Static Pod

> **API Server가 Pod라면, 그 Pod는 누가 만들었나?**
> Pod를 만들려면 API Server에 요청해야 하는데, API Server가 아직 없다.

**답: kubelet은 API Server 없이도 Pod를 띄울 수 있다.**

```bash
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

**kubelet이 이 디렉토리를 직접 감시한다.**

```
[일반 Pod]
kubectl → API Server → etcd → scheduler → kubelet → 컨테이너

[Static Pod]
/etc/kubernetes/manifests/*.yaml → kubelet → 컨테이너
                                    ↑ API Server 안 거침!
```

#### 부팅 순서

```
① systemd가 kubelet 시작
② kubelet이 /etc/kubernetes/manifests/ 스캔
③ etcd 컨테이너 실행
④ kube-apiserver 컨테이너 실행    ← 이제 API Server 존재!
⑤ kubelet이 API Server에 접속 → 노드 등록
⑥ scheduler, controller-manager 실행
```

**kubelet이 먼저, API Server가 나중이다.** 그래서 **kubelet만 유일하게 Pod가 아니라 systemd 서비스**다 — 자기가 Pod를 띄우는 존재라 스스로 Pod가 될 수 없다.

#### Static Pod의 흔적

```bash
kubectl get pods -n kube-system
# kube-apiserver-docker-desktop
#                └─ 노드 이름이 접미사 ★ Static Pod의 특징

kubectl delete pod kube-apiserver-docker-desktop -n kube-system
# → 즉시 되살아남! kubelet이 manifests를 보고 다시 만든다
# 진짜로 지우려면 YAML 파일을 지워야 함 (etcd가 아니라 디스크가 진짜 소스)

kubectl get pod kube-apiserver-docker-desktop -n kube-system \
  -o jsonpath='{.metadata.ownerReferences}'
# → kind: Node  ★ 일반 Pod는 ReplicaSet인데 Static Pod는 Node
```

#### API Server는 hostNetwork를 쓴다

```yaml
spec:
  hostNetwork: true    # 자기 net_ns를 안 만들고 호스트 것을 그대로 사용
```

```bash
kubectl get pods -n kube-system -o wide
# kube-apiserver-...   IP: 192.168.65.3   ← 노드 IP!
# coredns-...          IP: 10.244.0.5     ← Pod 대역
```

**왜?** Pod 네트워크(CNI)는 API Server가 떠 있어야 설정되는데, API Server가 Pod 네트워크를 기다리면 또 순환이다. **호스트 네트워크를 쓰면 그 의존성이 사라진다.**

---

### 10-6. kubectl vs kubelet, kubectl delete vs kind delete

#### kubectl vs kubelet — 위치가 정반대

```
[내 노트북]
  kubectl  ──HTTPS──> [API Server]
                          ↕ (watch)
                      [노드의 kubelet] ──> containerd ──> 컨테이너
```

| | **kubectl** | **kubelet** |
|---|---|---|
| 뜻 | kube **control** | kube **let** (데몬) |
| 어디서 | **내 컴퓨터** | **각 노드** |
| 정체 | CLI 프로그램 (내가 실행) | 상시 데몬 (systemd) |
| 역할 | 명령 **보내기** | 명령 **실행하기** |
| 실행 시점 | 내가 칠 때만 | **항상** |
| 설정 | `~/.kube/config` | `/var/lib/kubelet/config.yaml` |
| 개수 | 사람마다 하나 | **노드당 하나** |

> **비유**: kubectl = **리모컨**, kubelet = **에어컨 실외기**.

#### kubectl delete vs kind delete — 층이 다르다

```
┌─ kind/minikube/Docker Desktop: 이 층을 다룸 ─┐
│  클러스터 (노드, API Server, etcd)            │
│  ┌─ kubectl: 이 층을 다룸 ─────────────────┐ │
│  │  Pod, Deployment, Service...            │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

| | `kind delete cluster` | `kubectl delete pod x` |
|---|---|---|
| 대상 | **클러스터 전체** | 오브젝트 하나 |
| 통신 | Docker API 직접 (k8s 안 거침) | **API Server에 HTTP** |
| 복구 | `kind create cluster` (빈 클러스터) | ReplicaSet 있으면 **자동 재생성** |

> **비유**: `kubectl delete`는 **가구를 버리는 것**, `kind delete`는 **집을 철거하는 것**.

**`kubectl delete cluster`는 존재하지 않는다.**
- `Cluster`라는 kind가 없다 (클러스터는 논리적 개념이지 오브젝트가 아니다)
- kubectl은 API Server에 요청하는 도구인데, 클러스터를 지우려면 **API Server 자신을 죽여야** 한다 → 자기 발밑 자르기

**클러스터는 만든 도구가 지운다:**

| 만든 방법 | 지우는 방법 |
|---|---|
| `kind create cluster` | `kind delete cluster` |
| `minikube start` | `minikube delete` |
| **Docker Desktop** | 설정 → Kubernetes → **Reset Kubernetes Cluster** |
| `kubeadm init` | `kubeadm reset` + 수동 정리 |
| AWS EKS | `eksctl delete cluster` |

```bash
# 직접 구축한 클러스터 정리
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /etc/kubernetes /var/lib/etcd
sudo iptables -F && sudo iptables -t nat -F   # ← kube-proxy 규칙 청소 (필수!)
rm -rf ~/.kube
```

---

## 11장. 설정 분리 — ConfigMap / Secret

### 문제: 설정을 이미지에 박으면 안 되는 이유

```yaml
# application.yml — jar 안에 그대로 들어감
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/basecamp
    password: mySecret123
```

| 문제 | 상황 |
|---|---|
| ① **환경마다 재빌드** | dev/staging/prod DB가 달라 이미지를 3개 만들어야 함 |
| ② **설정 하나에 재배포** | 로그 레벨만 바꾸는데 빌드→푸시→배포 풀코스 |
| ③ **비밀번호 유출** | 이미지 레이어에 평문. `docker history`로 노출 |

> **원칙**: 이미지는 **어느 환경에서든 똑같아야 한다.** 환경 차이는 **밖에서 주입**한다.
> (12-Factor App 3번 원칙 = Spring profile을 쓰는 이유와 동일)

---

### 11-1. ConfigMap — 설정값 보관함

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:                          # 그냥 키-값 맵
  SPRING_PROFILES_ACTIVE: "prod"
  DB_HOST: "mysql-service"
  LOG_LEVEL: "INFO"
```

```bash
kubectl create configmap my-app-config \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=LOG_LEVEL=INFO

kubectl create configmap my-app-config --from-file=application.yml
```

---

### 11-2. Secret — 비밀번호 보관함 (⚠️ 암호화 아님)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret
type: Opaque
stringData:                    # 평문으로 쓰면 알아서 인코딩됨
  DB_PASSWORD: "mySecret123"
  JWT_SECRET: "abc123xyz"
```

#### ★ 반드시 알아야 할 것: Secret은 Base64 인코딩일 뿐이다

```bash
kubectl get secret my-app-secret -o yaml
#   DB_PASSWORD: bXlTZWNyZXQxMjM=      ← 암호처럼 보이지만

echo "bXlTZWNyZXQxMjM=" | base64 -d
# mySecret123                           ← 1초 만에 복호화
```

**Base64는 암호화가 아니라 "바이너리를 텍스트로 표현하는 방식"이다.** 열쇠가 필요 없고 누구나 디코딩할 수 있다.

> **Java 힌트**: `Base64`는 `java.security`가 아니라 `java.util`에 있다. 보안 도구가 아니라 유틸리티라는 뜻.

**그럼 Secret의 존재 의미는?**
- ConfigMap과 역할을 분리해 **RBAC 권한을 따로 부여** 가능
- `kubectl describe` 시 값이 안 찍힘 (실수 방지)
- etcd 저장 시 **암호화를 별도로 켤 수 있음** (EncryptionConfiguration)
- 로그·이벤트에 노출되지 않음

**실무**: AWS Secrets Manager, HashiCorp Vault를 External Secrets Operator로 연동해 진짜 암호화를 한다. k8s Secret만 믿으면 안 된다.

---

### 11-3. Pod에 주입하는 법 2가지

#### 방식 A: 환경변수 (가장 많이 씀)

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    envFrom:                        # 통째로 다 넣기
    - configMapRef:
        name: my-app-config
    - secretRef:
        name: my-app-secret
    env:                            # 하나만 골라 넣기
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-app-secret
          key: DB_PASSWORD
```

#### ★ Spring Boot Relaxed Binding — 코드 수정 0

Spring Boot는 환경변수를 자동으로 프로퍼티로 변환한다.

```
환경변수  SPRING_DATASOURCE_URL
            ↓ 언더스코어 → 점, 대문자 → 소문자
프로퍼티  spring.datasource.url
```

```yaml
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:mysql://mysql-service:3306/basecamp"
  SPRING_DATASOURCE_USERNAME: "appuser"
  SPRING_JPA_HIBERNATE_DDL_AUTO: "validate"
  LOGGING_LEVEL_ROOT: "INFO"
```

**앱 코드는 한 줄도 안 고쳐도 된다.** `application.yml`이 비어 있어도 위 환경변수만 있으면 DataSource가 붙는다.

**Spring Boot 설정 우선순위:**
```
1. 커맨드라인 인자
2. 환경변수                    ← ConfigMap/Secret이 여기
3. application-{profile}.yml
4. application.yml
```
jar 안에 기본값을 두고 **k8s에서 덮어쓰는** 구조가 자연스럽게 만들어진다.

> `jdbc:mysql://mysql-service:3306` — IP가 아니라 **Service 이름**을 쓴다. CoreDNS가 찾아준다. (7장)

#### 방식 B: 볼륨 마운트

```yaml
spec:
  containers:
  - name: my-app
    volumeMounts:
    - name: config-volume
      mountPath: /config          # 이 경로에 파일로 나타남
  volumes:
  - name: config-volume
    configMap:
      name: my-app-config
```

ConfigMap의 **키 = 파일명, 값 = 파일 내용**이 된다.

```yaml
env:
- name: SPRING_CONFIG_ADDITIONAL_LOCATION
  value: "file:/config/"
```

#### ★ 선택 기준 — 가장 중요한 함정

| | 환경변수 | 볼륨 마운트 |
|---|---|---|
| 적합 | 짧은 키-값 | 파일 통째로 (yml, 인증서, nginx.conf) |
| **변경 시 반영** | ❌ **Pod 재시작 필요** | ✅ 자동 갱신 (약 1분 지연) |
| 앱에서 읽기 | 자동 (Spring 바인딩) | 파일 읽기 로직 필요 |

**ConfigMap을 수정해도 환경변수로 주입한 Pod는 안 바뀐다.** 환경변수는 프로세스 시작 시 한 번 복사되는 값이기 때문.

```bash
kubectl rollout restart deployment/my-app   # 수정 후 필수
```

> **Java 비유**: `static final`로 초기화 때 값을 복사해둔 것. 원본이 바뀌어도 복사본은 그대로. 볼륨 마운트는 매번 파일을 다시 읽는 것.

---

### 11-4. 전체 그림

```
┌─ ConfigMap ─────────┐   ┌─ Secret ──────────┐
│ SPRING_PROFILES...  │   │ DB_PASSWORD       │
│ SPRING_DATASOURCE_..│   │ JWT_SECRET        │
└──────────┬──────────┘   └────────┬──────────┘
           │      envFrom / valueFrom
           └───────────┬────────────┘
                       ▼
              ┌─────────────────┐
              │  Deployment     │
              └────────┬────────┘
                       ▼
                ┌────┬────┬────┐
                │Pod │Pod │Pod │  ← 환경변수로 설정 주입
                └────┴────┴────┘
```

**환경별 분리:**
```
같은 이미지 my-app:1.0
  ├─ dev 네임스페이스     + ConfigMap(dev)     → dev DB
  ├─ staging 네임스페이스 + ConfigMap(staging) → staging DB
  └─ prod 네임스페이스    + ConfigMap(prod)    → prod DB
```
**이미지는 하나, 설정만 다름.** 이것이 목표였던 그림.

---

## 12장. Ingress — 클러스터의 L7 대문

### 문제: Service만으론 외부 노출이 비싸다

```
api.basecamp.com   → Service(LoadBalancer) → 클라우드 LB 1개 ($$$)
admin.basecamp.com → Service(LoadBalancer) → 클라우드 LB 1개 ($$$)
shop.basecamp.com  → Service(LoadBalancer) → 클라우드 LB 1개 ($$$)
```

서비스마다 LB 하나씩. 비용도, IP도, 인증서 관리 지점도 3배.

### 해결: 입구를 하나로 합치고 규칙으로 분배

```
                    ┌─ LoadBalancer 1개 ─┐
  모든 도메인  ────→ │  Ingress Controller │
                    └──────────┬──────────┘
                               │ 규칙 보고 분배
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      Service(api)     Service(admin)    Service(shop)
              │                │                │
            [Pod]            [Pod]            [Pod]
```

> **Spring 비유**: Ingress는 **`@RequestMapping` 라우팅 규칙표**를 애플리케이션이 아니라 **클러스터 입구에서** 적용하는 것. 또는 **Nginx 리버스 프록시 설정을 k8s 오브젝트화한 것** (실제 구현체가 대부분 nginx).

### Service vs Ingress — 계층이 다르다

| 오브젝트 | OSI 계층 | 아는 것 |
|---|---|---|
| **Service** | L4 (전송) | IP, 포트만 |
| **Ingress** | L7 (응용) | **도메인, URL 경로, HTTP 헤더** |

Service는 "10.96.0.42:80으로 온 패킷"까지만 안다. **URL을 모른다.** Ingress는 HTTP를 뜯어보므로 `/api/users`인지 `/admin`인지 구분한다.

### ★ Ingress는 두 조각이다 (핵심 함정)

| | 정체 | 역할 |
|---|---|---|
| **Ingress (리소스)** | YAML 규칙표 | "이 도메인은 저 Service로"라고 **적어둔 것** |
| **Ingress Controller** | 실제 도는 Pod (nginx 등) | 규칙을 읽고 **실제로 트래픽 처리** |

**Controller를 설치하지 않으면 Ingress YAML을 만들어도 아무 일도 일어나지 않는다.**

> **Java 비유**: Ingress = `interface`, Controller = 구현체. `List` 선언만으론 동작하지 않고 `ArrayList`를 넣어야 한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### YAML 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basecamp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx        # 어느 Controller가 처리할지
  rules:
  - host: api.basecamp.com       # ① 도메인으로 분기
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service   # ← 8장에서 만든 Service
            port:
              number: 80
      - path: /orders            # ② 경로로 분기
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
  tls:                            # ③ HTTPS를 여기서 종료
  - hosts:
    - api.basecamp.com
    secretName: basecamp-tls      # ← 인증서는 Secret에 (11장)
```

### ★ 전체 트래픽 흐름 (문서 전체가 여기서 만난다)

```
브라우저: https://api.basecamp.com/users
   ↓ DNS
클라우드 LoadBalancer (외부 IP)
   ↓
Ingress Controller Pod (nginx)
   ↓ TLS 종료 + Host/Path 규칙 매칭          ← 12장
Service: user-service (ClusterIP 10.96.0.42)
   ↓ CoreDNS 조회 + iptables DNAT            ← 7장 (br_netfilter 필요! 2장)
Pod (10.244.2.11:8080)                       ← 8장, 라벨로 연결
   ↓ 환경변수로 주입된 DB 접속 정보          ← 9장
Spring Boot 앱
```

### Ingress가 대신 해주는 일

| 기능 | 내용 |
|---|---|
| **TLS 종료** | HTTPS를 여기서 풀고 내부는 HTTP (인증서 한 곳만 관리) |
| **가상 호스팅** | 도메인별 분기 |
| **경로 라우팅** | `/api` → A, `/admin` → B |
| **rate limiting** | annotation으로 요청 제한 |
| **CORS** | annotation으로 헤더 처리 |
| **sticky session** | 같은 사용자를 같은 Pod로 |

> **Spring 앱 입장**: HTTPS 인증서 설정, CORS 필터, rate limit 필터를 코드에서 뺄 수 있다. 앱은 순수 비즈니스 로직만.

---

## 13장. 전체 연결 지도

이 문서의 모든 내용이 어떻게 하나로 이어지는가.

```
[0장] 컨테이너 = 호스트 커널을 빌려 쓰는 프로세스
   ↓ 그래서 호스트 커널을 미리 세팅해야 함
[2장] overlay + br_netfilter + sysctl 3종
   ↓ 이게 깔려야
[3~4장] Pod들이 노드를 넘나들며 통신 가능 (오버레이 네트워크)
   ↓ 그 위에서
[7장] Service + CoreDNS로 이름 기반 통신
   ↓ 그 Service가 찾아갈 Pod들을 관리하는 게
[8장] Deployment → ReplicaSet → Pod (라벨로 Service와 연결)
   ↓ 이 오브젝트들의 공통 문법이
[9장] metadata(신원) / spec(의도) / status(현실), Namespace와 Label
   ↓ 이걸 실제로 실행하는 주체가
[10장] API Server · etcd · Scheduler · Controller / kubelet · kube-proxy
   ↓ 그 Pod에 환경별 설정을 밖에서 주입
[11장] ConfigMap / Secret (Spring Relaxed Binding)
   ↓ 외부 트래픽이 들어오는 입구
[12장] Ingress (L7 라우팅 + TLS 종료)
   ↓ 이 모든 걸 조작하는 도구가
[5장] kubectl (+ context로 어느 클러스터인지 지정)
   ↓ 한편 자원 관리 측면에서
[6장] cgroup이 층마다 따로 걸림 → JVM 튜닝 시 주의
```

### 7장과 8장의 관계

같은 Service를 **두 각도**에서 본 것이다.

| | 7장 관점 | 8장 관점 |
|---|---|---|
| 질문 | "패킷이 **어떻게** 전달되나" | "Service가 Pod를 **어떻게 찾나**" |
| 답 | CoreDNS → ClusterIP → iptables DNAT | 라벨 셀렉터 매칭 → Endpoints |
| 층 | 네트워크 구현 | 오브젝트 관계 |

두 개가 만나는 지점이 **Endpoints**다. 라벨로 매칭된 Pod IP 목록이 Endpoints에 기록되고, kube-proxy가 그걸 읽어 iptables 규칙을 만든다.

```
Pod 라벨 매칭 → Endpoints 갱신 → kube-proxy가 감지 → iptables 규칙 재작성
```

### 명령어 ↔ 개념 매핑

| 2장의 명령 | 필요한 전제 지식 | 관련 장 |
|---|---|---|
| `overlay` 모듈 | 컨테이너 이미지가 **레이어로 쌓인다** | 0장 |
| `br_netfilter` 모듈 | 컨테이너가 **가상 브리지**로 연결된다 | 4장 |
| `bridge-nf-call-iptables=1` | k8s Service가 **iptables로 로드밸런싱**한다 | 7장 |
| `ip_forward=1` | Pod IP ≠ 노드 IP, **노드가 라우터 역할** | 4장 |

---

## 14장. 면접 대비 Q&A

<details>
<summary><b>Q1. 컨테이너와 VM의 차이를 설명하세요.</b></summary>

VM은 하이퍼바이저 위에서 **자체 커널을 가진 가짜 컴퓨터**로, Intel VT-x 같은 하드웨어 가상화로 격리됩니다. 컨테이너는 **호스트 커널을 공유하는 격리된 프로세스**로, 커널의 namespace(시야 격리)와 cgroup(자원 제한)으로 구현됩니다. 컨테이너가 훨씬 가볍고 빠르지만, 커널을 공유하므로 격리 수준은 VM보다 약합니다.
</details>

<details>
<summary><b>Q2. k8s 설치 전에 왜 커널 모듈과 sysctl을 건드리나요?</b></summary>

컨테이너가 호스트 커널을 그대로 쓰기 때문입니다. `overlay`는 컨테이너 이미지 레이어를 겹쳐 쌓는 파일시스템이고, `br_netfilter`는 가상 브리지를 지나는 패킷도 iptables가 검사하게 해줍니다. 후자가 없으면 kube-proxy의 Service 로드밸런싱이 동작하지 않습니다. `ip_forward=1`은 노드가 자기 것이 아닌 Pod 패킷을 다른 노드로 중계하는 라우터 역할을 하도록 켜는 것입니다.
</details>

<details>
<summary><b>Q3. Pod는 왜 존재하나요? 컨테이너를 바로 띄우면 안 되나요?</b></summary>

같은 Pod 안의 컨테이너들은 **IP와 볼륨을 공유**합니다. 그래서 서로를 `localhost`로 호출할 수 있고, 로그 수집기나 프록시 같은 사이드카 패턴을 자연스럽게 구성할 수 있습니다. Pod는 스케줄링과 IP 할당의 최소 단위이기도 합니다.
</details>

<details>
<summary><b>Q4. Pod IP는 어느 범위에서 유일한가요?</b></summary>

**클러스터 전체**에서 유일합니다. 노드가 달라도 겹치지 않습니다. 이것이 k8s 네트워크 모델의 대전제 중 하나이며, CNI 플러그인이 이를 보장합니다.
</details>

<details>
<summary><b>Q5. 노드가 다른 Pod끼리 어떻게 통신하나요?</b></summary>

Pod 네트워크는 노드 네트워크 위에 얹힌 **오버레이**입니다. VXLAN 방식이면 Pod 패킷을 노드 IP가 적힌 봉투로 한 번 더 감싸서(캡슐화) 보내고, 받는 노드가 겉봉투를 벗겨 Pod에 전달합니다. 이 과정에서 노드가 라우터 역할을 하므로 `ip_forward`가 켜져 있어야 합니다.
</details>

<details>
<summary><b>Q6. kubectl context가 뭔가요?</b></summary>

`클러스터(어디로) + 유저(누구로) + 네임스페이스(어느 방에서)` 세 가지를 한 세트로 묶어 이름 붙인 것입니다. kubectl은 API 서버로 HTTP 요청을 보내는 클라이언트일 뿐이라 목적지와 인증 정보를 알아야 하는데, 이를 `~/.kube/config`에 저장해두고 이름으로 전환하는 구조입니다. Spring의 profile과 같은 개념입니다.
</details>

<details>
<summary><b>Q7. 컨테이너에서 Java 앱이 OOMKilled 되는 고전적 원인은?</b></summary>

JVM이 힙 기본값을 정할 때 `/proc/meminfo`를 읽는데, 이건 컨테이너의 cgroup 제한이 아니라 **호스트(또는 VM) 전체 메모리**입니다. 그래서 512MB 제한 컨테이너에서 8GB 기준으로 힙을 2GB까지 잡으려다 커널에 죽습니다. `-XX:+UseContainerSupport`(Java 8u191+, 10+ 기본 활성화)가 cgroup 파일을 직접 읽게 해서 해결하며, 실무에선 `-XX:MaxRAMPercentage`로 비율 지정을 권장합니다.
</details>

<details>
<summary><b>Q8. k8s의 서비스 디스커버리는 어떻게 동작하나요?</b></summary>

두 겹의 간접 계층입니다. ① **CoreDNS**가 Service 이름을 ClusterIP로 변환하고, ② **kube-proxy가 심은 iptables 규칙**이 ClusterIP 패킷을 가로채 실제 Pod IP로 DNAT합니다. Pod가 죽고 새로 떠도 iptables 규칙만 갱신되고 Service IP와 이름은 그대로라, 호출하는 쪽은 변경을 몰라도 됩니다.
</details>

<details>
<summary><b>Q9. Spring Cloud Eureka와 k8s Service의 차이는?</b></summary>

역할은 같지만 계층이 다릅니다. Eureka는 **애플리케이션 레벨**에서 라이브러리가 서비스 목록을 관리하고 클라이언트 사이드 LB를 합니다. k8s Service는 **인프라 레벨**에서 CoreDNS와 커널 iptables가 처리합니다. 그래서 k8s에선 언어·프레임워크 무관하게 `http://service-name` 으로 호출만 하면 되고 별도 의존성이 필요 없습니다.
</details>

<details>
<summary><b>Q10. Headless Service는 언제 쓰나요?</b></summary>

`clusterIP: None`으로 설정하면 ClusterIP를 만들지 않고 DNS 조회 시 개별 Pod IP 목록을 반환합니다. 로드밸런싱 없이 각 인스턴스에 직접 접근해야 하는 DB 클러스터나 Kafka 같은 상태 저장 애플리케이션에서 StatefulSet과 함께 사용합니다.
</details>

<details>
<summary><b>Q11. Deployment, ReplicaSet, Pod의 관계는?</b></summary>

Deployment가 ReplicaSet을 만들고, ReplicaSet이 Pod를 만드는 3단 소유 구조입니다. ReplicaSet은 지정된 개수를 유지하는 역할만 하고, Deployment는 새 버전용 ReplicaSet을 추가로 만들어 양쪽 개수를 조절하며 롤링 업데이트를 수행합니다. 리소스 이름도 `my-app` → `my-app-7d9f8b` → `my-app-7d9f8b-abc12` 처럼 계층이 드러납니다.
</details>

<details>
<summary><b>Q12. 롤백이 어떻게 즉시 가능한가요?</b></summary>

Deployment가 이전 ReplicaSet을 삭제하지 않고 `replicas=0`으로만 남겨두기 때문입니다. `kubectl rollout undo`는 옛 ReplicaSet의 개수를 다시 올리고 현재 것을 0으로 내리는 동작입니다. 기본적으로 10개의 리비전 히스토리를 보관합니다.
</details>

<details>
<summary><b>Q13. Service는 Deployment와 직접 연결되어 있나요?</b></summary>

아닙니다. Service는 **라벨 셀렉터로 Pod를 찾을 뿐**이고 Deployment의 존재를 모릅니다. 그래서 Deployment를 삭제해도 Service는 남아 있고, 라벨만 일치하면 다른 워크로드의 Pod도 붙습니다. Endpoints가 비어 있다면 대부분 라벨 불일치가 원인입니다.
</details>

<details>
<summary><b>Q14. 선언형(Declarative) 방식이 무엇인가요?</b></summary>

"무엇을 해라"가 아니라 "어떤 상태여야 한다"를 선언하는 방식입니다. 에어컨을 켜라가 아니라 24도를 유지하라고 말하는 것과 같습니다. k8s의 컨트롤러들은 선언된 desired state와 현재 상태를 계속 비교해 차이를 메우는 루프를 돌며, ReplicaSet의 개수 유지가 대표적인 예입니다.
</details>

<details>
<summary><b>Q15. 컨테이너의 "격리"가 정확히 무엇인가요?</b></summary>

커널이 프로세스가 **볼 수 있는 범위(namespace)** 와 **쓸 수 있는 양(cgroup)** 을 잘라주는 것입니다. namespace는 PID·NET·MNT·UTS·IPC·USER·CGROUP 7종류로 자원 종류별로 시야를 차단하며, 안 보이면 건드릴 수도 없으므로 시야 차단 자체가 보호막이 됩니다. cgroup은 시야는 그대로 두고 사용량 상한만 겁니다.
</details>

<details>
<summary><b>Q16. `-p 8080:80`에서 두 포트는 각각 어디 소속인가요?</b></summary>

왼쪽 8080은 **호스트 NET namespace**의 포트, 오른쪽 80은 **컨테이너 NET namespace**의 포트입니다. 서로 다른 포트 공간이므로 여러 컨테이너가 모두 내부 80을 써도 충돌하지 않지만, 호스트의 8080은 하나뿐이라 중복하면 `port is already allocated` 에러가 납니다. `-p`는 두 namespace 사이에 iptables DNAT 규칙으로 다리를 놓는 옵션입니다.
</details>

<details>
<summary><b>Q17. 같은 Pod의 컨테이너들이 localhost로 통신 가능한 이유는?</b></summary>

**NET namespace를 공유**하기 때문입니다. IP도 포트 공간도 하나이므로 서로를 localhost로 부를 수 있고, 반대로 같은 Pod 안에서 두 컨테이너가 같은 포트를 쓰면 충돌합니다. k8s는 pause 컨테이너로 namespace를 먼저 만들고 나머지를 합류시키며, Docker의 `--network=container:이름` 과 같은 원리입니다.
</details>

<details>
<summary><b>Q18. VM의 격리는 컨테이너와 어떻게 다른가요?</b></summary>

VM은 **CPU 하드웨어**가 격리를 강제합니다. Intel VT-x/AMD-V가 VMX Root/Non-root 모드를 만들어 게스트의 특권 명령을 VM Exit으로 낚아채고, EPT/NPT라는 하드웨어 주소 변환표가 게스트 메모리를 물리적으로 분리합니다. VM마다 완전한 커널이 따로 부팅되며, 그래서 VM 안에서 다시 컨테이너를 돌릴 수 있습니다.
</details>

<details>
<summary><b>Q19. k8s Secret은 안전한가요?</b></summary>

기본 상태로는 안전하지 않습니다. Secret은 암호화가 아니라 **Base64 인코딩**일 뿐이라 누구나 즉시 디코딩할 수 있습니다. 의미가 있는 부분은 ConfigMap과 역할을 분리해 RBAC를 따로 걸 수 있고, etcd 저장 시 암호화를 별도로 활성화할 수 있으며, describe나 로그에 값이 노출되지 않는다는 점입니다. 실무에서는 Vault나 AWS Secrets Manager를 연동합니다.
</details>

<details>
<summary><b>Q20. ConfigMap을 수정했는데 앱에 반영되지 않는 이유는?</b></summary>

환경변수로 주입한 경우 값이 **프로세스 시작 시 한 번 복사**되기 때문입니다. `kubectl rollout restart deployment/이름` 으로 재시작해야 반영됩니다. 볼륨 마운트 방식은 파일이 자동 갱신되지만 약 1분의 지연이 있고, 앱이 파일을 다시 읽는 로직을 갖고 있어야 합니다.
</details>

<details>
<summary><b>Q21. Service가 있는데 Ingress가 왜 필요한가요?</b></summary>

Service는 L4라서 IP와 포트만 알고 URL을 모릅니다. 도메인이나 경로로 분기하려면 L7이 필요합니다. 또 서비스마다 LoadBalancer를 만들면 비용과 인증서 관리 지점이 서비스 수만큼 늘어나는데, Ingress는 입구를 하나로 합쳐 TLS 종료·가상 호스팅·경로 라우팅을 한 곳에서 처리합니다.
</details>

<details>
<summary><b>Q22. Ingress YAML을 만들었는데 동작하지 않습니다.</b></summary>

**Ingress Controller가 설치되지 않았을 가능성**이 높습니다. Ingress 리소스는 규칙을 적어둔 선언일 뿐이고, 실제로 트래픽을 처리하는 것은 nginx 등으로 구현된 Controller Pod입니다. 인터페이스만 있고 구현체가 없는 상태와 같습니다. `ingressClassName`이 설치된 Controller와 일치하는지도 확인해야 합니다.
</details>

<details>
<summary><b>Q23. 커널이 하는 일이 무엇인가요?</b></summary>

CPU·메모리·디스크·네트워크 카드라는 하드웨어를 대신 관리하는 계층입니다. 사용자 프로그램은 CPU의 Ring 3에서만 실행되어 하드웨어를 직접 만질 수 없고, **시스템 콜(syscall)** 이라는 유일한 창구로 커널에 요청해야 합니다. 이때 CPU가 Ring 0으로 전환되어 커널이 작업을 처리합니다.
</details>

<details>
<summary><b>Q24. namespace가 왜 필요한지 근본 원인을 설명하세요.</b></summary>

커널이 관리하는 프로세스 목록, 포트 테이블, 마운트 테이블 같은 **전역 자료구조가 하나뿐**이기 때문입니다. 그래서 앱 두 개를 한 서버에 띄우면 포트가 겹치고 서로의 프로세스가 보입니다. namespace는 이 전역 목록을 **여러 벌로 복제**하는 기능으로, Java로 치면 static 필드를 인스턴스 필드로 바꾼 것과 같습니다. 목록 종류마다 하나씩 있어서 PID·NET·MNT·UTS·IPC·USER·CGROUP 7종류가 존재합니다.
</details>

<details>
<summary><b>Q25. cgroup의 'c'는 무엇이며 어떻게 구현되어 있나요?</b></summary>

**Control**입니다. Control Group, 즉 프로세스를 묶어서 자원 사용을 제어한다는 뜻입니다. 구현은 `/sys/fs/cgroup` 아래의 디렉토리 트리이며, `memory.max` 같은 파일에 제한값을 쓰고 `cgroup.procs`에 PID를 써서 프로세스를 편입시킵니다. 트리 구조인 이유는 부모의 한도가 자식들의 합을 덮어 강제하기 위해서입니다.
</details>

<details>
<summary><b>Q26. `/proc`과 `/sys`의 차이는?</b></summary>

둘 다 디스크가 아닌 커널이 즉석에서 만드는 가상 파일시스템입니다. `/proc`은 **프로세스 자신에 대한 정보**를 조회하는 곳으로 대부분 읽기 전용이고, `/sys`는 **커널 객체를 조작하는 인터페이스**로 값을 써서 동작을 바꿀 수 있습니다. 컨테이너에서 `/sys/fs/cgroup`은 자기 것만 보이도록 마운트되지만 `/proc/meminfo`는 호스트 것이 노출되며, 이 불일치가 JVM OOMKilled의 원인입니다.
</details>

<details>
<summary><b>Q27. pause 컨테이너는 왜 필요한가요?</b></summary>

Pod의 NET namespace를 붙잡아두는 홀더입니다. namespace는 참조 카운트가 0이 되면 소멸하는데, 앱 컨테이너가 주인이면 크래시할 때마다 namespace가 사라져 IP가 바뀝니다. pause가 참조를 유지하므로 앱이 재시작해도 Pod IP가 보존됩니다. 그 외에도 컨테이너가 여러 개일 때 **누가 namespace 주인인지 정하는 문제**를 없애고, `shareProcessNamespace`를 켰을 때 PID 1로서 좀비를 수거하며, Pod 레벨 cgroup을 살아있게 붙잡습니다.
</details>

<details>
<summary><b>Q28. pause와 Service는 역할이 겹치지 않나요?</b></summary>

지켜주는 구간이 다릅니다. pause는 **Pod가 살아있는 동안** IP를 유지하고, Service는 **Pod가 죽어도** 접근 주소를 유지합니다. pause는 Pod와 함께 죽으므로 Pod 재생성·롤링 업데이트·스케일링 앞에선 무력하고, 그 구간을 Service가 iptables와 DNS로 덮습니다. pause는 실제 프로세스이고 Service는 실체가 없는 추상화라는 점도 다릅니다.
</details>

<details>
<summary><b>Q29. init 컨테이너가 만든 파일을 메인 컨테이너가 어떻게 읽나요?</b></summary>

**볼륨을 통해서만** 가능합니다. 같은 Pod라도 MNT namespace는 공유하지 않아 파일시스템이 완전히 독립적이기 때문입니다. `emptyDir` 같은 볼륨을 양쪽 컨테이너가 각각 마운트하면, 볼륨은 컨테이너가 아니라 **Pod에 속하므로** init 컨테이너가 종료된 뒤에도 내용이 남아 메인 컨테이너가 읽을 수 있습니다.
</details>

<details>
<summary><b>Q30. Deployment가 Pod를 "관리한다"는 게 구체적으로 무엇인가요?</b></summary>

`selector`로 라벨이 일치하는 Pod를 **세고**, `replicas`와 비교해 모자라면 `template`으로 **만들고** 넘치면 **지우는** 루프입니다. 중요한 점은 자기가 만든 Pod를 기억하지 않고 **매번 라벨로 다시 판단**한다는 것입니다. 그래서 Pod의 라벨을 떼면 그 Pod는 고아가 되고 Deployment는 부족분을 새로 만듭니다. 자가 치유와 스케일링 모두 이 루프의 부산물입니다.
</details>

<details>
<summary><b>Q31. labels는 metadata에, selector는 spec에 있는 이유는?</b></summary>

labels는 **"나는 이런 이름표를 달고 있다"** 는 자기 신원이라 모든 오브젝트가 공유하는 `metadata`에 들어가고, selector는 **"나는 이런 대상을 관리하고 싶다"** 는 원하는 상태라 오브젝트마다 다른 `spec`에 들어갑니다. 이 배치 덕분에 API Server가 오브젝트 종류와 무관하게 라벨을 인덱싱할 수 있고, `kubectl get all -l app=web` 같은 종류 무관 검색이 가능합니다.
</details>

<details>
<summary><b>Q32. k8s Namespace와 Linux namespace의 차이는?</b></summary>

이름만 같은 완전히 다른 것입니다. Linux namespace는 **프로세스**에 적용되는 커널 자료구조로 진짜 격리를 수행하고, k8s Namespace는 **오브젝트**에 붙는 etcd의 문자열 필드로 이름 충돌 방지와 RBAC·Quota 구획이 목적입니다. 특히 **k8s Namespace는 네트워크를 격리하지 않습니다** — 다른 네임스페이스의 Service에 그냥 통신되며, 막으려면 NetworkPolicy가 필요합니다.
</details>

<details>
<summary><b>Q33. API Server도 Pod인데, 그 Pod는 누가 만들었나요?</b></summary>

**Static Pod** 메커니즘입니다. kubelet은 `/etc/kubernetes/manifests/` 디렉토리를 직접 감시해서 API Server 없이도 컨테이너를 띄울 수 있습니다. 부팅 시 systemd → kubelet → etcd·API Server 순으로 올라가고, 그 다음 kubelet이 API Server에 사후 등록합니다. 이 때문에 **kubelet만 유일하게 Pod가 아니라 systemd 서비스**이며, Static Pod는 이름에 노드명이 붙고 소유자가 ReplicaSet이 아닌 Node로 기록됩니다.
</details>

<details>
<summary><b>Q34. Service는 왜 L7 라우팅을 할 수 없나요?</b></summary>

Service의 실체가 커널 netfilter의 iptables 규칙이기 때문입니다. 커널은 성능상 HTTP를 파싱하지 않으므로 IP와 포트까지만 볼 수 있고, URL 경로나 Host 헤더는 패킷 페이로드 안에 있어 접근할 수 없습니다. 그래서 도메인·경로 분기나 TLS 종료가 필요하면 사용자 공간에서 도는 nginx 기반의 Ingress Controller가 필요합니다.
</details>

---

## 15장. 실습 체크리스트

### 커널 레벨 확인 (서장 실습)
```bash
# 1. syscall 직접 보기 — 프로그램이 커널에 무엇을 부탁하는가
strace ls 2>&1 | head -20

# 2. 내가 속한 namespace 목록
ls -l /proc/self/ns/

# 3. 내가 속한 cgroup 경로
cat /proc/self/cgroup

# 4. cgroup은 그냥 폴더다
ls /sys/fs/cgroup/
cat /sys/fs/cgroup/memory.max

# 5. ★ 도커 없이 namespace 만들어보기
sudo unshare --pid --net --mount --uts --fork bash
  hostname mycontainer   # 호스트에 영향 없음
  ps aux                 # 자기 것만 보임
  ip addr                # lo만 있음
  exit

# 6. ★ /proc과 /sys의 불일치 = JVM OOMKilled의 원인
docker run --rm --memory=512m alpine sh -c '
  echo "[/sys] 내 제한:"; cat /sys/fs/cgroup/memory.max
  echo "[/proc] 호스트 메모리:"; head -1 /proc/meminfo
'
```
> **5번과 6번이 핵심.** 5번은 컨테이너가 마법이 아님을, 6번은 두 가상 파일시스템의 성격 차이를 몸으로 보여준다.

### 커널 설정 확인
```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

### context 확인
```bash
kubectl config get-contexts
kubectl config current-context
kubectl config view
```

### 계층 구조 눈으로 보기
```bash
kubectl get nodes -o wide          # 노드 이름 + INTERNAL-IP
kubectl get pods -o wide           # Pod IP + 어느 노드에 떠 있는지
kubectl get svc                    # Service의 CLUSTER-IP
kubectl get endpoints              # Service ↔ 실제 Pod IP 매핑 ★
```
> `get endpoints`가 특히 중요하다. **Service 이름 뒤에 실제로 어떤 Pod IP들이 붙어 있는지** 직접 보여준다.

### cgroup 불일치 체험
```bash
docker run --rm --memory=512m alpine cat /sys/fs/cgroup/memory.max
docker run --rm --memory=512m alpine free -m
# 두 값이 다른 것을 확인 → JVM OOMKilled의 원인
```

### DNS 동작 확인
```bash
kubectl run test --rm -it --image=busybox --restart=Never -- sh
# 컨테이너 안에서:
cat /etc/resolv.conf
nslookup kubernetes.default
```

---

### 워크로드 오브젝트 실습 (5분 코스)
```bash
# 1. Deployment 생성
kubectl create deployment my-app --image=nginx --replicas=3

# 2. 족보 확인
kubectl get deploy,rs,pod

# 3. Pod 하나 죽여보기 ★ 하이라이트
kubectl delete pod <pod이름>
kubectl get pod          # 즉시 새 Pod 생성됨 = ReplicaSet이 일한 것

# 4. Service 붙이기
kubectl expose deployment my-app --port=80

# 5. 연결 확인
kubectl get endpoints my-app

# 6. 롤링 업데이트
kubectl set image deployment/my-app nginx=nginx:1.25
kubectl rollout status deployment/my-app

# 7. 롤백
kubectl rollout undo deployment/my-app
kubectl rollout history deployment/my-app

# 8. 정리
kubectl delete deployment my-app
kubectl delete service my-app
```
> **3번을 반드시 해볼 것.** Pod를 지웠는데 몇 초 뒤 새 Pod가 생기는 걸 눈으로 보면 선언형 개념이 몸으로 이해된다.

### ConfigMap / Secret 실습
```bash
# 1. 생성
kubectl create configmap demo-config \
  --from-literal=LOG_LEVEL=DEBUG \
  --from-literal=APP_NAME=basecamp
kubectl create secret generic demo-secret \
  --from-literal=DB_PASSWORD=mypass123

# 2. ★ Base64가 암호화가 아님을 직접 확인
kubectl get secret demo-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
echo
# → mypass123 그대로 출력됨

# 3. Pod에 주입해서 확인
kubectl run test --image=busybox --restart=Never -it --rm \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "test", "image": "busybox",
      "command": ["sh"], "stdin": true, "tty": true,
      "envFrom": [
        {"configMapRef": {"name": "demo-config"}},
        {"secretRef": {"name": "demo-secret"}}
      ]
    }]
  }
}'
# 컨테이너 안에서: env | grep -E 'LOG_LEVEL|APP_NAME|DB_PASSWORD'

# 4. 정리
kubectl delete configmap demo-config
kubectl delete secret demo-secret
```
> **2번을 반드시 해볼 것.** Secret이 암호화가 아님을 손으로 확인하는 것이 이 주제의 핵심.

### Ingress 실습
```bash
# 1. Controller 설치 (Docker Desktop)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx     # Running 될 때까지 대기

# 2. 앱 두 개 준비
kubectl create deployment app-a --image=nginx
kubectl create deployment app-b --image=httpd
kubectl expose deployment app-a --port=80
kubectl expose deployment app-b --port=80

# 3. Ingress 생성
kubectl create ingress demo \
  --class=nginx \
  --rule="localhost/a*=app-a:80" \
  --rule="localhost/b*=app-b:80"

# 4. 확인 — 하나의 입구, 경로별 다른 응답
kubectl get ingress
curl http://localhost/a    # nginx 페이지
curl http://localhost/b    # Apache 페이지

# 5. 정리
kubectl delete ingress demo
kubectl delete deployment app-a app-b
kubectl delete service app-a app-b
```

### Pod 내부 구조 실습
```bash
# ★ 네트워크는 공유, 파일시스템은 독립
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: {name: two}
spec:
  containers:
  - {name: web, image: nginx}
  - {name: helper, image: busybox, command: ['sh','-c','sleep 3600']}
EOF

kubectl exec two -c helper -- wget -qO- localhost:80 | head -3  # ✅ 통함
kubectl exec two -c web    -- ls /usr/share/nginx               # 있음
kubectl exec two -c helper -- ls /usr/share/nginx               # 없음!
kubectl delete pod two

# init 컨테이너 단계 관찰
kubectl get pod init-web -w            # Init:0/1 → PodInitializing → Running
kubectl logs init-web -c create-index  # 죽은 뒤에도 로그 조회 가능

# pause 확인 (WSL2)
docker ps -a | grep pause
```
> **마지막 두 줄이 핵심.** `localhost`는 통하는데 파일은 안 보이는 조합이 Pod의 정체를 정확히 보여준다.

### 라벨과 Namespace 실습
```bash
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --port=80

# 1. Pod에 붙은 라벨 = template.metadata.labels
kubectl get pods --show-labels
kubectl get deploy web -o jsonpath='{.spec.template.metadata.labels}'

# 2. ★★ 라벨을 떼면 — Pod가 4개가 된다
POD=$(kubectl get pod -l app=web -o name | head -1)
kubectl label $POD app-
kubectl get pods --show-labels     # 고아 1 + 정상 2 + 새로 생긴 1
kubectl get endpoints web          # 고아는 빠짐

# 3. Deployment를 지워도 고아는 살아남음
kubectl delete deployment web
kubectl get pods
kubectl delete pod --all
kubectl delete svc web

# 4. Namespace는 이름 유일성의 범위
kubectl create ns dev && kubectl create ns prod
kubectl run web --image=nginx -n dev
kubectl run web --image=nginx -n prod
kubectl get pods -A | grep web     # 같은 이름 공존 ✅

# 5. ★ k8s Namespace는 네트워크를 격리하지 않는다
PRODIP=$(kubectl get pod web -n prod -o jsonpath='{.status.podIP}')
kubectl exec -n dev web -- ping -c1 $PRODIP    # 통함!

# 6. Namespace 삭제 = 안의 모든 것 삭제
kubectl delete ns dev prod
```
> **2번과 5번이 핵심.** 2번은 "관리 = 라벨 매칭"을, 5번은 "k8s Namespace ≠ 격리"를 증명한다.

### 아키텍처 실습
```bash
# 1. Control Plane 컴포넌트도 노드 위의 Pod다
kubectl get pods -n kube-system -o wide

# 2. ★ Static Pod는 지워도 부활
kubectl delete pod -n kube-system kube-apiserver-docker-desktop
sleep 5 && kubectl get pods -n kube-system | grep apiserver

# 3. Static Pod의 소유자는 ReplicaSet이 아니라 Node
kubectl get pod -n kube-system kube-apiserver-docker-desktop \
  -o jsonpath='{.metadata.ownerReferences}'

# 4. hostNetwork — API Server는 Pod 대역이 아닌 노드 IP
kubectl get pods -n kube-system -o wide | grep -E 'apiserver|coredns'

# 5. 컨트롤 플레인은 그냥 라벨일 뿐
kubectl get nodes --show-labels | tr ',' '\n' | grep node-role
```

### 네트워크 계층 실습
```bash
# 테이블 3형제를 나란히
echo "=== 라우팅 (어느 방향) ==="; ip route
echo "=== iptables (조작 규칙) ==="; sudo iptables -t nat -L -n | head -20
echo "=== 소켓 (누구에게) ==="; ss -tlnp

# ★ Service의 실체 = DNAT 규칙 몇 줄
kubectl create deployment web --image=nginx && kubectl expose deployment web --port=80
CIP=$(kubectl get svc web -o jsonpath='{.spec.clusterIP}')
sudo iptables -t nat -L -n | grep $CIP
# → 목적지 IP:포트만. URL/도메인 조건이 없음 = Service가 L4인 이유
kubectl delete deployment web && kubectl delete svc web
```

---

## 다음 학습 방향

| 우선순위 | 주제 | 이유 |
|---|---|---|
| 1 | **Probe (liveness / readiness / startup)** | 롤링 업데이트의 "준비되면" 판단 기준. Spring Actuator와 직결 |
| 2 | **리소스 requests / limits + JVM 튜닝** | 0-3장·6장의 실전 적용. OOMKilled 예방 |
| 3 | **볼륨 / PV / PVC / StatefulSet** | DB 등 상태 저장 워크로드 |
| 4 | **HPA (오토스케일링)** | replicas를 부하에 따라 자동 조절 |
| 5 | **Helm** | YAML 폭증 관리. 템플릿 + 값 분리 |
| 6 | **리눅스 네트워크 실습** — 브리지·veth pair 직접 만들기 | 4장이 손에 안 잡히면 여기가 빈 곳 |