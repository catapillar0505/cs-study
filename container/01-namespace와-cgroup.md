> [← container 목차](README.md)

# namespace와 cgroup — 컨테이너의 재료

## S-5. ★ 그래서 namespace가 왜 필요한가

문제는 **이 목록들이 전부 하나뿐**이라는 것.

```
전역 프로세스 목록  1개  → 모두가 남의 프로세스를 봄
전역 포트 테이블   1개  → 8080은 세상에 하나뿐
전역 마운트 테이블  1개  → 모두가 같은 / 를 봄
```

한 서버에 앱 두 개를 띄우면 충돌하는 근본 원인이다. **공유 자원이 하나뿐이라서.**

### 해결책 두 가지

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

### "시야 차단"이라는 표현의 진짜 의미

컨테이너 A의 프로세스는 **목록 #2만 참조**한다. 목록 #1이 존재하는지도 모른다.

```bash
docker run --rm alpine ps aux
# PID 1  ← 목록 #2에는 이것뿐이라 1번
```

**숨긴 게 아니라, 애초에 다른 목록을 보고 있는 것.**

### 왜 7종류인가

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

## S-6. cgroup — 이름부터

> **cgroup = Control Group = 제어 그룹**
> `Control(제어) + Group(묶음)` = **"프로세스를 묶어서 자원 사용을 제어한다"**

### namespace로는 안 되는 것

namespace는 **보이는 것을 자를 뿐** 자원을 못 막는다.

```
컨테이너 A: 자기 프로세스만 보임 (namespace ✅)
          하지만 메모리를 8GB 다 먹을 수 있음 ❌
          → 컨테이너 B가 OOM으로 죽음
```

**시야를 잘라도 물리 자원은 여전히 하나를 나눠 쓴다.** 그래서 별도 기능이 필요했다.

### 작동 방식 — 전부 파일 조작

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

**트리인 이유**: 부모 한도가 자식들을 덮기 위해서. 자식 합이 5GB여도 부모가 4GB면 4GB에서 막힌다. ([cgroup 중첩 구조](03-cgroup-중첩과-jvm-oomkilled.md)의 "VM 8GB 제한이 이긴다"가 이 원리)

### 그럼 CGROUP namespace는?

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

## S-7. namespace와 cgroup은 완전히 별개다

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

## S-8. /proc vs /sys — 신분증 vs 조종실

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

> `/proc/sys/` 는 `/proc` 안에 있지만 성격은 `/sys`에 가깝다. sysfs가 나중에 생기면서 남은 **역사적 잔재**이며, `sysctl`이 만지는 곳이 여기다. (→ [커널 파라미터와 모듈](../os/linux/12-커널-파라미터와-sysctl.md))

### ★ 이 차이가 JVM OOMKilled의 원인이다

도커는 컨테이너의 cgroup 디렉토리를 `/sys/fs/cgroup`으로 **바인드 마운트**해준다 (MNT namespace). 하지만 **`/proc/meminfo`는 가려주지 않는다.**

```bash
docker run --rm --memory=512m alpine sh -c '
  cat /sys/fs/cgroup/memory.max   # 536870912  = 512MB (정확)
  head -1 /proc/meminfo           # 8GB        = 호스트 것 (부정확) ★
'
```

JVM이 하필 뒤엣것을 읽어 힙을 2GB로 잡다가 죽었다. `-XX:+UseContainerSupport`가 하는 일이 **읽는 경로를 `/proc`에서 `/sys/fs/cgroup`으로 바꾸는 것**이다. (→ [cgroup 중첩 구조](03-cgroup-중첩과-jvm-oomkilled.md))

---

## S-9. 컨테이너 = 조립품

이제 컨테이너의 정체가 보인다. **새로운 기술이 아니다.**

```
컨테이너 = namespace(격리)
         + cgroup(자원 제한)
         + overlayfs(이미지 레이어)
         + capabilities(권한 축소)
         + seccomp(syscall 필터)
```

**이미 있던 커널 기능들의 조합.** 도커가 한 일은 이걸 `docker run` 한 줄로 포장한 것이다.

### 도커 없이 손으로 만들어보기

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

## S-10. 이 문서의 개념들이 커널 어디에 붙는가

| 개념 | 커널상의 위치 | 등장 장 |
|---|---|---|
| `modprobe overlay` | **커널 모듈** = 기능을 나중에 끼우는 것 | [커널 파라미터](../os/linux/12-커널-파라미터와-sysctl.md) |
| `br_netfilter` | 커널 모듈 (netfilter 관련) | [커널 파라미터](../os/linux/12-커널-파라미터와-sysctl.md) |
| `sysctl` | 커널 **동작 스위치** (`/proc/sys/`) | [커널 파라미터](../os/linux/12-커널-파라미터와-sysctl.md) |
| Pod가 IP 공유 | **NET namespace 공유** | [컨테이너는 VM이 아니다](02-컨테이너는-vm이-아니다.md) |
| 포트 충돌 없음 | NET namespace가 포트 테이블 복제 | [컨테이너는 VM이 아니다](02-컨테이너는-vm이-아니다.md) |
| 컨테이너별 메모리 제한 | **cgroup** | [컨테이너는 VM이 아니다](02-컨테이너는-vm이-아니다.md) |
| JVM OOMKilled | cgroup은 제한하나 `/proc`은 안 가림 | [cgroup 중첩 구조](03-cgroup-중첩과-jvm-oomkilled.md) |
| VM 격리 | **하이퍼바이저 + CPU Ring/EPT** | [컨테이너는 VM이 아니다](02-컨테이너는-vm이-아니다.md) |

---

[컨테이너는 VM이 아니다 →](02-컨테이너는-vm이-아니다.md)
