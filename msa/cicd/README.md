# cicd — CI/CD 자동화

서비스가 여러 개로 늘어나면 **손으로 배포하는 것이 불가능해집니다.**
코드를 Push하면 **빌드·테스트·이미지 생성·배포까지 기계가 알아서** 하도록 만드는 방법을 정리합니다.

특히 쿠버네티스 환경의 표준이 된 **GitOps** — **"Git에 적힌 상태가 곧 클러스터의 상태"** 라는
발상을 중심으로 두 가지 구성을 직접 만들어 봅니다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [CI/CD란](01-ci-cd란.md) | 수동 배포의 한계, CI가 푸는 문제, Delivery와 Deployment의 차이, 파이프라인 |
| 02 | [GitOps](02-gitops.md) | 선언적이란 무엇인가, Single Source of Truth, **Push vs Pull**, Drift와 자가 치유 |
| 03 | [ArgoCD](03-argocd.md) | 아키텍처와 컴포넌트, **Helm 기초**, Application 리소스, 설치, 상태 읽는 법 |
| 04 | [실습 1 — GitHub Actions와 ArgoCD](04-실습-github-actions와-argocd.md) | 저장소 하나 구성, 멀티 스테이지 빌드, `sed`·`[skip ci]`, 무한 루프 방지 |
| 05 | [Jenkins](05-jenkins.md) | SaaS와의 차이, Controller/Agent, Jenkinsfile, Credentials, Webhook, DooD |
| 06 | [실습 2 — Jenkins와 ArgoCD](06-실습-jenkins와-argocd.md) | **저장소 분리**, 멀티 아키텍처 빌드, 터널링(ngrok), 두 실습 비교 |

## 전체 그림

```
 개발자
   │ git push
   ▼
[ Git: 애플리케이션 코드 ]
   │ 트리거
   ▼
[ CI ]  GitHub Actions 또는 Jenkins
   │
   ├─ 빌드 · 테스트
   ├─ 컨테이너 이미지 생성 → [ 레지스트리 ]  (태그 = 빌드 번호)
   └─ 매니페스트의 이미지 태그를 수정해 Git 에 커밋   ★ CI 와 CD 를 잇는 다리
   ▼
[ Git: 배포 매니페스트 ]
   ▲
   │ 변경 감지 (Pull)
   │
[ CD ]  ArgoCD
   │ 동기화
   ▼
[ 쿠버네티스 클러스터 ]  → 롤링 업데이트
```

**CI는 클러스터를 전혀 건드리지 않습니다.** Git에 커밋만 하고, 나머지는 ArgoCD가 합니다.
그래서 **CI 서버가 운영 클러스터의 열쇠를 가질 필요가 없습니다.**

## 이 폴더를 관통하는 문장들

> **Git 저장소의 상태 = 실제 클러스터의 상태.** Git이 유일한 진실 공급원이 된다.

> **Push는 밖에서 명령을 밀어 넣고, Pull은 안에서 스스로 가져간다.**
> 후자가 안전한 이유는 **클러스터 밖으로 권한을 내보내지 않기 때문**이다.

> **모든 빌드에 고유한 태그를 붙여라.** `latest`를 쓰면 어느 코드가 배포됐는지 알 수 없고 롤백도 안 된다.

> **위험한 건 자동화가 아니라 검증 없는 자동화다.** 손으로 하는 배포가 오히려 더 위험하다.

## 읽는 순서

배경지식이 없다면 **01 → 02 → 03**으로 개념을 잡고, **04 실습**을 한 번 돌려 보세요.
Jenkins가 필요 없다면 **05·06은 건너뛰어도** 됩니다 — 다만 **저장소 분리 이야기**는 읽어둘 값어치가 있습니다.

```
01 CI/CD 란       "왜 자동화하나"
   ↓
02 GitOps         "누가 배포를 시작하나"     ← 이 폴더의 핵심 발상
   ↓
03 ArgoCD         "그걸 하는 도구"
   ↓
04 실습 1          가장 단순한 구성 (저장소 하나)
   ↓
05 Jenkins        "직접 구축하는 CI"
   ↓
06 실습 2          실무형 구성 (저장소 분리)
```

## 함께 보기

- 컨테이너 이미지와 태그 전략 → [container/06](../../container/06-이미지-태그-전략.md)
- 레지스트리와 인증(PAT) → [container/04](../../container/04-이미지-레지스트리.md) · [container/05](../../container/05-레지스트리-인증.md)
- 배포되는 대상(Deployment·Service) → [kubernetes/05](../../kubernetes/05-워크로드-deployment-replicaset-service.md)
- 롤링 업데이트와 프로브 → [kubernetes/11](../../kubernetes/11-노드-rollout-프로브.md)
- 설정과 비밀값 관리 → [kubernetes/08](../../kubernetes/08-configmap과-secret.md)
- 인프라 자체를 코드로 관리하기 → [devops](../../devops/README.md)
- 배포한 뒤 잘 돌아가는지 보기 → [observation](../observation/README.md)
