# monitoring — 인프라 운영 모니터링

서버·클러스터·클라우드 계정을 **운영하는 입장에서** 무엇을 수집하고, 언제 사람을 깨우고, 장애 뒤에 무엇을 남길지 정리합니다.

[msa/observation](../../msa/observation/README.md)이 **애플리케이션 관점**(Trace ID로 지표·추적·로그 잇기, PLG 실습)이라면,
이 폴더는 **인프라 관점**(노드·클러스터·AWS 계정, 알림 설계, 장애 대응 절차)입니다. 겹치는 개념은 링크만 겁니다.

출발점은 제 프로젝트([Micro-Lens](https://github.com/MICRO-LENS/microlens-infra) · [메추라기](https://github.com/catapillar0505/mechuragi_infra))를 운영하며 다음으로 갖추고 싶어진 것들입니다.

- **메트릭을 추이로 보기** — 현재값이 아니라 며칠치 흐름으로 판단하기
- **로그를 한곳에 모으기** — 파드·서버가 바뀌어도 남는 로그
- **사람을 깨우는 기준 세우기** — 증상 기반 알림과 런북
- **GPU·노드 지표 상시 수집** — 수동 측정 대신 exporter로

## 목차 (작성 예정)

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | 모니터링 설계 원칙 | 골든 시그널, USE·RED 방법, 모니터링 vs 관측 가능성 |
| 02 | CloudWatch 메트릭과 알람 | 기본 메트릭에 메모리가 없는 이유, Agent 메트릭, Metric Filter, Alarm + SNS |
| 03 | CloudWatch Logs Insights | 로그 그룹 설계, 보존 기간, Insights 쿼리, 로그 비용 |
| 04 | CloudTrail · Config · GuardDuty | 감사 로그, 설정 변경 이력과 드리프트 감지, 위협 탐지 |
| 05 | kube-prometheus-stack | Prometheus·Alertmanager·Grafana·node-exporter·kube-state-metrics, Helm + ArgoCD 설치 |
| 06 | exporter 정리 | node · mysql · redis · blackbox · DCGM exporter가 각각 무엇을 수집하나 |
| 07 | Loki로 K8s 로그 수집 | Promtail/Alloy DaemonSet, 라벨 설계, LogQL |
| 08 | 알림 설계 | 증상 기반 알림, grouping·inhibition, 알림 피로 |
| 09 | SLO와 에러 버짓 | SLI·SLO·SLA 차이, 에러 버짓 계산과 배포 판단 |
| 10 | 런북과 포스트모템 | 템플릿, 트러블슈팅 기록을 포스트모템 형식으로 바꾸기 |
| 11 | 실습 — Micro-Lens에 적용 | kube-prometheus-stack + Loki를 ArgoCD로 배포, 노드 1대 중지 → 알림 수신까지 |

## 함께 보기

- 지표·추적·로그의 개념과 PLG 실습 → [msa/observation](../../msa/observation/README.md)
- 응답 시간을 어떤 숫자로 볼 것인가 → [performance/01 p95와 백분위수](../../performance/01-p95와-백분위수.md)
- 프로브와 롤아웃 → [kubernetes/11](../../kubernetes/11-노드-rollout-프로브.md)
- 코드 밖 변경(drift) → [devops/terraform/01](../terraform/01-상태-파일과-잠금.md)
