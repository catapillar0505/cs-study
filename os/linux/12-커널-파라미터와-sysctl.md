> [← Linux 목차](README.md) · [os](../README.md)

# 커널 파라미터와 모듈 — sysctl

리눅스 커널의 동작을 조절하는 설정값들입니다. 커널을 다시 컴파일하지 않고 실행 중에 바꿀 수 있습니다.

보는 방법이 두 가지인데 같은 것을 가리킵니다. `/proc/sys/` 아래 파일로 직접 읽고 쓸 수도 있고, sysctl 명령어로도 됩니다.

파일로 바꾸면 재부팅 시 사라집니다. 영구 적용하려면 `/etc/sysctl.conf`나 `/etc/sysctl.d/` 아래에 적어둬야 합니다. 이건 실무에서 자주 실수하는 지점입니다. 지금 고쳐서 문제가 해결됐는데 재부팅하면 다시 발생하는 경우입니다.

예를 들면, vm.dirty_ratio와 vm.dirty_background_ratio는 메모리에 쌓인 미반영 쓰기 데이터를 언제 디스크로 내려보낼지 정하는 값입니다. net.netfilter.nf_conntrack_max는 연결 추적 테이블의 최대 크기입니다.

쿠버네티스를 설치할 때도 몇 개를 반드시 건드립니다. net.bridge.bridge-nf-call-iptables를 켜야 파드 트래픽이 iptables 규칙을 타고, net.ipv4.ip_forward를 켜야 노드가 패킷을 전달할 수 있습니다. 이걸 안 하면 클러스터는 뜨는데 통신이 안 되는 상태가 됩니다.


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

**왜 문제인가**: k8s는 **kube-proxy가 iptables 규칙으로 Service 로드밸런싱**을 한다. Pod끼리 브리지 통신할 때 iptables를 안 거치면 **서비스 라우팅이 통째로 깨진다.** (→ [서비스 디스커버리](../../kubernetes/04-서비스디스커버리-service와-coredns.md))

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

[← CFS 스케줄러와 nice](11-cfs-스케줄러와-nice.md) | [리눅스 — 헷갈리는 함정과 약자 정리 →](99-함정과-약자-정리.md)
