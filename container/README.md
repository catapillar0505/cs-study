# container — 컨테이너

컨테이너는 가벼운 VM이 아니라 **namespace로 시야를 자르고 cgroup으로 자원을 묶은 프로세스**다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [namespace와 cgroup](01-namespace와-cgroup.md) | 시야 격리 vs 자원 격리, /proc와 /sys, 컨테이너 = 조립품 |
| 02 | [컨테이너는 VM이 아니다](02-컨테이너는-vm이-아니다.md) | namespace 7종, overlay 파일시스템, VM 격리와의 차이 |
| 03 | [cgroup 중첩과 JVM OOMKilled](03-cgroup-중첩과-jvm-oomkilled.md) | 호스트·게스트 이중 제한, JVM이 힙을 잘못 잡는 이유 |
| 04 | [이미지 레지스트리](04-이미지-레지스트리.md) | 사내 레지스트리를 두는 이유, 배치에 따른 트래픽 경로 |
| 05 | [프라이빗 레지스트리 인증](05-레지스트리-인증.md) | imagePullSecrets vs credential provider, 메타데이터 v1/v2와 SSRF |
| 06 | [이미지 태그 전략](06-이미지-태그-전략.md) | latest가 위험한 이유, 커밋 해시 태그, 불변 태그 |
