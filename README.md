# cs-study

공부한 CS 개념을 영역별로 정리해 둔 저장소입니다. 문서 127편.

수업을 들으며 남긴 기록에서 시작해, 나중에 다시 찾아볼 수 있도록 다듬었습니다.
처음에는 날짜별로 쌓다 보니 한 파일에 여러 주제가 뒤섞였고, 그래서 **주제 단위로 다시 쪼개고 영역별로 나눴습니다.**
지금은 한 파일에 한 주제만 있고, 각 폴더의 `README.md`가 그 영역의 목차 역할을 합니다.

정리하면서 지킨 기준이 몇 가지 있습니다.

- **"무엇"보다 "왜 그렇게 동작하는지"를 남긴다.** 명령어 사용법보다, 그 명령이 커널에서 무슨 일을 일으키는지를 적었습니다.
- **헷갈렸던 지점을 그대로 남긴다.** 처음에 잘못 이해했던 것, 이름이 비슷해서 섞어 쓰던 것들을 "흔한 오해" 항목으로 따로 뒀습니다.
- **계층을 따라 아래에서 위로 쌓는다.** 하드웨어 → 리눅스 커널 → 네트워크·스토리지 → 가상화 → 컨테이너 → 쿠버네티스 순서로, 아래층을 알아야 위층이 이해되도록 배치했습니다.
- **한 개념이 여러 영역에 걸치면 각 영역에 두고 서로 링크한다.** NAT은 원리(network)와 클라우드 구현(aws)에, 컨테이너는 커널 기능(os)과 컨테이너 관점(container)에 나눠 두고 연결했습니다.

인프라 쪽(운영체제·네트워크·스토리지·가상화·컨테이너·쿠버네티스·클라우드)이 가장 두껍고,
백엔드 개발에서 쓰는 것들(Java·Spring·JPA·MySQL)과 프론트엔드 기초가 함께 있습니다.

## 디렉토리 구조

```
cs-study/
├── os/                            운영체제
│   ├── linux/               13    유닉스·POSIX, 부팅, 프로세스/systemd, 파일시스템·마운트,
│   │                              권한, 사용자·그룹, 셸 명령어, 시스템 콜, 인터럽트,
│   │                              CFS 스케줄러, 커널 파라미터(sysctl)
│   └── windows/              1    Active Directory, 그룹 정책(GPO), PowerShell
│
├── network/                  8    OSI 7계층, L2~L7, TCP/UDP, 가상 네트워크·오버레이,
│                                 패킷·MTU, NIC, NAT 커넥션 추적, TLS 종료, WebSocket/SSE
├── storage/                  5    DAS·NAS·SAN, 블록·파일·오브젝트, 리눅스 블록 계층,
│                                 SSD·NVMe, Ceph, EBS·EFS·S3
├── hardware/                 2    x86 아키텍처, 온프레미스 서버와 벤더
├── virtualization/           3    가상화 원리, 네 가지 자원 가상화, 하이퍼바이저 Type 1/2
├── container/                6    namespace·cgroup, 컨테이너 ≠ VM, cgroup 중첩과 JVM,
│                                 레지스트리와 인증, 이미지 태그 전략
├── kubernetes/              15    계층 구조, 네트워크, 서비스 디스커버리, 워크로드,
│                                 오브젝트 구조, 클러스터 아키텍처, ConfigMap/Secret,
│                                 Ingress, 스케줄링, 프로브·rollout, 실습·Q&A
│
├── cloud/                         클라우드
│   ├── public-cloud/              빌려 쓰는 클라우드
│   │   └── aws/             12    클라우드 5계층, CIDR, VPC·서브넷, 라우팅·IGW, NAT GW,
│   │                              SG·NACL, 트래픽 흐름, Bastion, 리전·엣지, 3Tier, HA
│   └── private-cloud/             직접 짓는 클라우드
│       └── open-stack/       1    퍼블릭과 무엇이 다른가, 표준화가 먼저인 이유
│
├── devops/                        자동화·IaC
│   ├── ansible/              2    동작 원리와 멱등성, playbook 작성 원칙
│   └── terraform/            1    상태 파일과 잠금, drift
│
├── database/                      데이터베이스
│   ├── basics/               3    관계형 DB 개념, DDL, DQL
│   ├── mysql/               11    스토리지 엔진, 트랜잭션·ACID, 이상 현상, 격리 수준,
│   │                              InnoDB(MVCC·락·로그), 인덱스, SARGable, EXPLAIN
│   └── redis/                2    키 만료와 이벤트, 캐시 직렬화 함정
│
├── ai/                       7    CNN, Attention, ONNX, ONNX Runtime 스레드, GPU 인프라,
│                                 모델 가중치 배포, 객체 탐지 임계값·NMS
├── performance/              3    p95와 백분위수, 스레드풀, 캐시 전략과 무효화
│
├── java/                     7    자료형, 상속과 다형성, JCF·제네릭, 람다, 입출력, 예외
├── spring/                  12    IoC/DI, 요청과 응답, RESTful API, AOP, MyBatis,
│                                 트랜잭션, Spring Security, 토큰 인증, 복습 퀴즈
├── jpa/                      6    영속성 컨텍스트, 연관관계 매핑, N+1 문제 (+ Q&A)
├── frontend/                      프론트엔드
│   ├── javascript/           2    개념 정리, 핵심 총정리
│   └── react/                3    Vite 시작하기, JSX·Babel 트러블슈팅, 개념 총정리
│
├── git/                      1    Git 기초
└── tools/                    1    마크다운 문법
```

## 영역별 목차

| 영역 | 목차 | 문서 |
|---|---|---|
| 운영체제 | [os](os/README.md) — [linux](os/linux/README.md) · [windows](os/windows/README.md) | 14 |
| 네트워크 | [network](network/README.md) | 8 |
| 스토리지 | [storage](storage/README.md) | 5 |
| 컴퓨터 구조 | [hardware](hardware/README.md) | 2 |
| 가상화 | [virtualization](virtualization/README.md) | 3 |
| 컨테이너 | [container](container/README.md) | 6 |
| 쿠버네티스 | [kubernetes](kubernetes/README.md) | 15 |
| 클라우드 | [cloud](cloud/README.md) — [aws](cloud/public-cloud/aws/README.md) · [open-stack](cloud/private-cloud/open-stack/README.md) | 13 |
| 자동화·IaC | [devops](devops/README.md) — [ansible](devops/ansible/README.md) · [terraform](devops/terraform/README.md) | 3 |
| 데이터베이스 | [database](database/README.md) — [basics](database/basics/README.md) · [mysql](database/mysql/README.md) · [redis](database/redis/README.md) | 16 |
| 인공지능 | [ai](ai/README.md) | 7 |
| 성능·측정 | [performance](performance/README.md) | 3 |
| 자바 | [java](java/README.md) | 7 |
| 스프링 | [spring](spring/README.md) | 12 |
| JPA | [jpa](jpa/README.md) | 6 |
| 프론트엔드 | [frontend](frontend/README.md) — [javascript](frontend/javascript/README.md) · [react](frontend/react/README.md) | 5 |
| 버전 관리 | [git](git/README.md) | 1 |
| 도구 | [tools](tools/README.md) | 1 |

## 읽는 법

- 문서마다 **맨 위에 목차 링크**, **맨 아래에 이전/다음 문서 링크**가 있습니다.
- 파일명은 `NN-주제.md` 형식이고, 번호가 그 영역 안에서 읽는 순서입니다.
- 아래에서 위로 쌓는 순서로 보려면 `os/linux` → `network`·`storage` → `virtualization` → `container` → `kubernetes` 순서를 권합니다.

## 영역이 이어지는 지점

한 주제가 여러 영역에 걸칠 때는 각 영역에 두고 서로 링크했습니다.

- **컨테이너의 원리** → 커널 기능이므로 [os/linux](os/linux/README.md)(시스템 콜·커널 파라미터)와 [container](container/README.md)(namespace·cgroup)로 나뉩니다
- **NAT** → 원리는 [network](network/README.md), 클라우드에서의 구현은 [aws](cloud/public-cloud/aws/README.md)
- **가상화** → 개념은 [virtualization](virtualization/README.md), 그 위의 컨테이너는 [container](container/README.md)
- **프라이빗 클라우드** → 개념은 [open-stack](cloud/private-cloud/open-stack/README.md), 구성 요소(Neutron·Ceph)는 [network](network/README.md)·[storage](storage/README.md)
- **캐시** → 동작은 [database/redis](database/redis/README.md), 설계 전략은 [performance](performance/README.md)
- **고가용성** → 설계 원칙은 [aws/12](cloud/public-cloud/aws/12-고가용성-설계.md), 스케줄링 구현은 [kubernetes/10](kubernetes/10-podantiaffinity와-스케줄링.md)
