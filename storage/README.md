# storage — 스토리지

저장장치를 어떻게 연결하고(DAS/NAS/SAN), 어떤 단위로 다루고(블록/파일/오브젝트), 커널이 어떤 경로로 쓰는지.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [연결 방식 — DAS · NAS · SAN](01-연결-방식-das-nas-san.md) | 세 방식의 차이, 파이버 채널, SAN 스위치, iSCSI |
| 02 | [블록 · 파일 · 오브젝트](02-블록-파일-오브젝트.md) | 접근 단위와 접근 방법, 어디에 무엇을 쓰는가 |
| 03 | [블록 계층과 SSD · NVMe](03-블록-계층과-ssd-nvme.md) | 페이지 캐시 → 파일시스템 → 블록 계층, NVMe 병렬성, aqu-sz |
| 04 | [Ceph — 분산 스토리지](04-ceph-분산-스토리지.md) | OSD/MON/MGR, CRUSH, 복제와 acting set, RBD/CephFS/RGW |
| 05 | [AWS 스토리지](05-aws-스토리지-ebs-efs-s3.md) | EBS · EFS · S3의 차이, 부팅 볼륨, AZ 제약 |
