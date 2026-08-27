# kubernetes — 쿠버네티스

계층 구조 → 네트워크 → 워크로드 → 아키텍처 순서로 읽으면 좋습니다.
컨테이너의 원리는 [container](../container/README.md), 커널 설정은 [os/linux](../os/linux/README.md)에 있습니다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [계층 구조](01-계층구조-컨테이너-pod-노드-클러스터.md) | 컨테이너/Pod/노드/클러스터, pause·init 컨테이너 |
| 02 | [네트워크 — 오버레이와 언더레이](02-네트워크-오버레이와-언더레이.md) | 주소 평면 2개, 캡슐화, netfilter/iptables, L4 vs L7 |
| 03 | [kubectl context와 kubeconfig](03-kubectl-context와-kubeconfig.md) | kubeconfig 구조, context 전환, 사고 방지 습관 |
| 04 | [서비스 디스커버리](04-서비스디스커버리-service와-coredns.md) | Service + CoreDNS, 간접 계층 2겹, Headless Service |
| 05 | [워크로드 오브젝트](05-워크로드-deployment-replicaset-service.md) | Deployment / ReplicaSet / Service, 롤링 업데이트와 롤백 |
| 06 | [오브젝트 구조](06-오브젝트구조-metadata-namespace-label.md) | metadata/spec/status, Label 셀렉터, Namespace vs Label |
| 07 | [클러스터 아키텍처](07-클러스터-아키텍처.md) | Control Plane / Worker 컴포넌트, apply의 흐름, Static Pod |
| 08 | [ConfigMap / Secret](08-configmap과-secret.md) | 주입 방법 2가지, Secret은 암호화가 아니다 |
| 09 | [Ingress](09-ingress.md) | L7 대문, Ingress는 두 조각, 전체 트래픽 흐름 |
| 10 | [podAntiAffinity와 스케줄링](10-podantiaffinity와-스케줄링.md) | 스케줄러 2단계, topologyKey, required의 Pending 함정 |
| 11 | [노드 · rollout · 프로브](11-노드-rollout-프로브.md) | 워커 노드, 롤링 업데이트, liveness/readiness, 이미지 pre-pull |
| 90 | [전체 연결 지도](90-전체-연결지도.md) | 문서 간 관계, 명령어 ↔ 개념 매핑 |
| 91 | [개념 확인 Q&A](91-개념-확인-qa.md) | 스스로 점검하는 질문과 답 |
| 92 | [실습 체크리스트](92-실습-체크리스트.md) | 커널·네트워크·워크로드 실습 명령 모음 |
| 93 | [관리형 vs 직접 구축](93-관리형-vs-직접구축.md) | EKS 등과 kubeadm의 차이, 직접 세울 때 드러나는 것 |
