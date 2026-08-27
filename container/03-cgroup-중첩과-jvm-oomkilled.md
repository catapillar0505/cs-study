> [← container 목차](README.md)

# cgroup 중첩 구조와 JVM OOMKilled

## 질문: 호스트가 VM에 cgroup 걸고, VM이 컨테이너에 또 거는가?

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

## 게스트는 호스트의 cgroup을 볼 수 없다

```bash
# VM 안에서
$ free -h
              total
Mem:          7.7Gi     ← "내 컴퓨터 RAM이 8GB구나"
```

게스트 커널은 이걸 **cgroup 제한이 아니라 하드웨어 스펙**으로 인식한다. 하이퍼바이저가 가상 하드웨어로 만들어 보여준 것이기 때문.

> 비유: 회사가 팀에 예산 8천만원 배정 → 팀장이 팀원에게 "너 1천만원까지" 제한. 팀원은 **회사 전체 예산이 얼마인지, 왜 우리 팀이 8천만원인지 알 수 없다.**

## 두 제한은 곱해지지 않고, 작은 쪽이 이긴다

```
컨테이너: "나 16GB 쓸래"   ← 게스트 cgroup은 허락
VM 전체:  8GB뿐            ← 호스트가 막음
결과: 8GB 근처에서 OOM
```

게스트 커널은 16GB를 못 준다는 걸 **미리 알 수 없다.** 실제로 요청하다 물리적으로 없어서 터진다.
→ 실무에선 컨테이너 제한의 총합이 VM 크기를 넘지 않게 설계.

## 내 환경 (Docker Desktop on Windows)

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

> **VM 안에 컨테이너가 들어있는 중첩 구조.** [커널 파라미터와 모듈](../os/linux/12-커널-파라미터와-sysctl.md) 설정도 Windows 커널이 아니라 **WSL2 리눅스 커널**에 적용된다.

## ★ 실전 payoff — JVM이 여기서 터진다

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

## 직접 확인

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

[← 컨테이너는 VM이 아니다](02-컨테이너는-vm이-아니다.md) | [이미지 레지스트리 — Harbor →](04-이미지-레지스트리.md)
