# private cloud — 프라이빗 클라우드

직접 짓고 운영하는 클라우드. 자원이 유한하고, 관리형으로 제공되던 것을 직접 세워야 하고,
장애 단위가 물리 단위(랙·전원)로 내려오는 환경입니다. 플랫폼별로 폴더를 나눕니다.

| 플랫폼 | 목차 | 문서 |
|---|---|---|
| [open-stack](open-stack/README.md) | 퍼블릭과의 차이, 표준화가 먼저인 이유 | 1편 |

## 다른 폴더에 있는 관련 문서

프라이빗 클라우드(OpenStack·Ceph) 구성 요소는 성격상 각 CS 영역에 두었습니다.

| 주제 | 문서 |
|---|---|
| 가상 네트워크 · 테넌트 · Floating IP · L3 에이전트 | [network/03](../../network/03-가상-네트워크와-테넌트.md) |
| Ceph 분산 스토리지 (OSD/MON, CRUSH, RBD·CephFS·RGW) | [storage/04](../../storage/04-ceph-분산-스토리지.md) |
| 스토리지 연결 방식 (DAS · NAS · SAN) | [storage/01](../../storage/01-연결-방식-das-nas-san.md) |
| 가상화와 하이퍼바이저 | [virtualization](../../virtualization/README.md) |
| 서버 하드웨어와 벤더 | [hardware/02](../../hardware/02-온프레미스-서버와-하드웨어.md) |
| 고가용성 설계 · 3Tier | [aws/12](../public-cloud/aws/12-고가용성-설계.md) · [aws/11](../public-cloud/aws/11-3tier-아키텍처.md) |
