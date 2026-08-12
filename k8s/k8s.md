# 쿠버네티스 기초 정리

> 커널 사전 설정부터 서비스 디스커버리까지 — 기초 → 심화 순서
>
> 다루는 범위: 컨테이너의 정체 / 리눅스 커널 준비 작업 / k8s 계층 구조 / 네트워크 2층 구조 / kubectl context / cgroup 중첩 / 서비스 디스커버리

---

## 목차

- [0장. 전제 지식 — 컨테이너는 VM이 아니다](#0장-전제-지식--컨테이너는-vm이-아니다)
- [1장. 쉘 문법 — heredoc과 tee](#1장-쉘-문법--heredoc과-tee)
- [2장. k8s 설치 전 커널 설정 4단계](#2장-k8s-설치-전-커널-설정-4단계)
- [3장. k8s 계층 구조 — 컨테이너 / Pod / 노드 / 클러스터](#3장-k8s-계층-구조--컨테이너--pod--노드--클러스터)
- [4장. 네트워크 — 오버레이와 언더레이](#4장-네트워크--오버레이와-언더레이)
- [5장. kubectl context와 kubeconfig](#5장-kubectl-context와-kubeconfig)
- [6장. cgroup 중첩 구조와 JVM OOMKilled](#6장-cgroup-중첩-구조와-jvm-oomkilled)
- [7장. 서비스 디스커버리 — Service + CoreDNS](#7장-서비스-디스커버리--service--coredns)
- [8장. 워크로드 오브젝트 — Deployment / ReplicaSet / Service](#8장-워크로드-오브젝트--deployment--replicaset--service)
- [9장. 전체 연결 지도](#9장-전체-연결-지도)
- [10장. 면접 대비 Q&A](#10장-면접-대비-qa)
- [11장. 실습 체크리스트](#11장-실습-체크리스트)

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

## 9장. 전체 연결 지도

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

## 10장. 면접 대비 Q&A

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

---

## 11장. 실습 체크리스트

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

---

## 다음 학습 방향

| 우선순위 | 주제 | 이유 |
|---|---|---|
| 1 | **ConfigMap / Secret** | Spring 설정 외부화와 직결 (다음 세션) |
| 2 | **Ingress** | 외부 트래픽 유입 경로 |
| 3 | **리소스 requests/limits + JVM 튜닝** | 6장의 실전 적용 |
| 4 | **Probe (liveness/readiness)** | 롤링 업데이트의 "준비되면" 판단 기준 |
| 5 | **리눅스 네트워크 실습** — 브리지·veth pair 직접 만들기 | 4장이 손에 안 잡히면 여기가 빈 곳 |