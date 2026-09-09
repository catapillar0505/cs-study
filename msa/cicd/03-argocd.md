> [← cicd 목차](README.md)

# ArgoCD

## 한 줄로

**클러스터 안에 사는 GitOps 에이전트**다. Git의 매니페스트와 실제 클러스터 상태를 **계속 비교**하다가, 다르면 맞춰 준다.

## 무엇을 하는 도구인가

> 쿠버네티스에서 [GitOps](02-gitops.md)를 구현하는 **선언적 CD 도구**

세 가지 일을 합니다.

| | 하는 일 |
|---|---|
| **비교(Diff)** | Git에 적힌 **원하는 상태(Desired State)** 와 클러스터의 **실제 상태(Live State)** 를 대조 |
| **동기화(Sync)** | 다르면 클러스터를 **Git 쪽에 맞춤** |
| **보여주기** | UI·CLI로 **무엇이 어떻게 다른지, 무엇이 배포됐는지** 표시 + 롤백 |

**Git 저장소를 Single Source of Truth로 지정하고, 거기서 눈을 떼지 않는 감시자**라고 보면 됩니다.

### 무엇을 읽을 수 있나

Git 저장소에 있는 **쿠버네티스 매니페스트**를 읽는데, 형식은 여러 가지를 지원합니다.

| 형식 | 설명 |
|---|---|
| **일반 YAML** | 그냥 `kubectl apply` 할 수 있는 평범한 매니페스트 |
| **Helm** | 템플릿 + 값 파일 구조 (아래에서 설명) |
| **Kustomize** | 기본 YAML에 **환경별 덮어쓰기**를 얹는 방식 |

## 기초 — Helm이란

이 실습에서 계속 나오므로 먼저 알아야 합니다.

**문제**: 같은 애플리케이션을 개발·스테이징·운영에 배포하는데 **replica 수와 이미지 태그만 다릅니다.**

```
   deployment-dev.yaml       replicas: 1,  tag: "dev"
   deployment-staging.yaml   replicas: 2,  tag: "1.2.0"
   deployment-prod.yaml      replicas: 10, tag: "1.2.0"
        ↑ 나머지 200줄은 완전히 똑같은데 파일이 3개 💀
```

**Helm은 이걸 "템플릿 + 값"으로 쪼갭니다.**

```
   templates/deployment.yaml     ← 뼈대 (값이 들어갈 자리를 비워둠)
   values.yaml                   ← 값만 따로
```

```yaml
# templates/deployment.yaml — 뼈대
spec:
  replicas: {{ .Values.replicaCount }}          # ← values.yaml 에서 가져옴
  ...
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```yaml
# values.yaml — 값
replicaCount: 2
image:
  repository: myid/myapp
  tag: "1"
```

**읽는 법 두 가지만 알면 됩니다.**

| 표기 | 뜻 |
|---|---|
| `.Values.xxx` | **`values.yaml` 파일**의 값을 가져옴 |
| `.Chart.xxx` | **`Chart.yaml` 파일**의 값을 가져옴 (차트 이름·버전 등) |

> **차트(Chart)** 는 **Helm이 다루는 패키지 단위**입니다. `Chart.yaml`이 그 이름표입니다.
>
> **`Chart.yaml`이 있으면 ArgoCD가 "아, Helm이구나" 하고 알아서 판단**해서,
> 내부적으로 템플릿을 렌더링한 뒤 클러스터에 적용합니다. **Helm을 따로 설치할 필요가 없습니다.**

**GitOps에서 Helm이 특히 유용한 이유**: CI가 **`values.yaml`의 이미지 태그 한 줄만 고치면**
배포가 트리거됩니다. 200줄짜리 매니페스트를 건드릴 필요가 없습니다.

## 아키텍처

```
         ┌───────────────────────────────────┐
         │           사용자 (UI / CLI)         │
         └─────────────────┬─────────────────┘
                           │ REST / gRPC
         ┌─────────────────▼─────────────────┐
         │          API SERVER               │
         │  (인증·권한, Webhook 처리)          │
         └─────────────────┬─────────────────┘
            ┌──────────────┴─────────────┐
┌───────────▼────────────┐     ┌─────────▼──────────┐
│ APPLICATION CONTROLLER │     │     REPO SERVER    │
│  (동기화 · 차이 감지)    │     │ (Git 클론 & 렌더링) │
└───────────┬────────────┘     └─────────┬──────────┘
            │ 조회 / 적용                  │ 매니페스트 생성
┌───────────▼────────────┐     ┌─────────▼──────────┐
│    쿠버네티스 API        │     │     Git 저장소      │
└────────────────────────┘     └────────────────────┘
```

| 컴포넌트 | 역할 |
|---|---|
| **API Server** | Web UI·CLI·Webhook 요청을 받고 **인증/권한(RBAC)** 처리 |
| **Application Controller** | **Git 상태와 클러스터 상태를 비교**하고 동기화 수행 — **핵심 엔진** |
| **Repo Server** | Git을 **로컬에 복제·캐싱**하고 Helm/Kustomize를 **최종 YAML로 변환** |
| **Redis** | 상태·이력 **캐싱**으로 성능 최적화 |

> **왜 Repo Server가 따로 있나?** Git 클론과 템플릿 렌더링은 **무겁고 느린 작업**이라
> 동기화 엔진과 분리해 두었습니다. 캐싱도 여기서 합니다.

## Application — ArgoCD의 핵심 리소스

ArgoCD에게 **"이 Git 경로를 저 클러스터에 맞춰 줘"** 라고 알려주는 설정이 **Application**입니다.

> 이것 자체도 **쿠버네티스 리소스**입니다. ArgoCD가 설치될 때 **CRD**로 등록됩니다.
>
> **CRD = Custom Resource Definition.**
> 쿠버네티스에 **원래 없던 새로운 리소스 종류를 추가**하는 기능입니다.
> `Pod`, `Service`처럼 `Application`도 `kubectl`로 다룰 수 있게 됩니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application                     # ← CRD 로 추가된 리소스 종류
metadata:
  name: my-app
  namespace: argocd                   # ArgoCD 가 사는 네임스페이스
spec:
  project: default

  # ① 어디에 있는 걸 배포하나 (Git)
  source:
    repoURL: 'https://github.com/<USER>/my-repo.git'
    targetRevision: main              # 추적할 브랜치
    path: k8s/helm                    # 저장소 안에서 매니페스트가 있는 경로

  # ② 어디로 배포하나 (클러스터)
  destination:
    server: 'https://kubernetes.default.svc'   # 같은 클러스터의 내부 주소
    namespace: default                          # 앱이 실제로 뜰 네임스페이스

  # ③ 어떻게 동기화하나
  syncPolicy:
    automated:
      prune: true                     # Git 에서 지우면 클러스터에서도 삭제
      selfHeal: true                  # 손으로 바꾸면 Git 상태로 되돌림
    syncOptions:
      - CreateNamespace=true          # 대상 네임스페이스가 없으면 만들어라
```

### 세 블록만 보면 됩니다

| 블록 | 질문 |
|---|---|
| **`source`** | **"어디 있는 걸?"** — Git 주소 + 브랜치 + 경로 |
| **`destination`** | **"어디로?"** — 클러스터 주소 + 네임스페이스 |
| **`syncPolicy`** | **"어떻게?"** — 자동인가 수동인가, 지울 건가 되돌릴 건가 |

### 헷갈리는 두 개의 namespace

```yaml
metadata:
  namespace: argocd      # ← 이 Application 설정 자체가 사는 곳 (= ArgoCD 가 있는 곳)
spec:
  destination:
    namespace: default   # ← 실제 애플리케이션(Pod, Service)이 뜨는 곳
```

**둘은 다릅니다.** ArgoCD는 `argocd` 네임스페이스에 살면서, 앱은 `default`에 배포하는 것입니다.

### `prune`과 `selfHeal`

| 옵션 | 켜면 | 끄면 |
|---|---|---|
| **`prune: true`** | Git에서 리소스를 지우면 **클러스터에서도 삭제** | Git에서 지워도 **클러스터에는 남아 있음** (유령 리소스) |
| **`selfHeal: true`** | `kubectl`로 손댄 것도 **Git 상태로 되돌림** | 손댄 상태가 **그대로 유지** (Drift 방치) |

> **둘 다 켜야 진짜 GitOps입니다.**
> 다만 **처음 도입할 때는 무섭습니다.** "Git에서 실수로 지웠는데 운영 리소스가 사라진다면?"
> 그래서 **자동 동기화를 끄고 수동 승인으로 시작**했다가, 익숙해지면 켜는 경우가 많습니다.

### `https://kubernetes.default.svc`가 뭔가

**쿠버네티스 내부에서 자기 자신의 API 서버를 부르는 고정 주소**입니다.

```
   kubernetes  .  default  .  svc
   서비스 이름     네임스페이스   서비스 접미사
```

→ [kubernetes/04 서비스 디스커버리](../../kubernetes/04-서비스디스커버리-service와-coredns.md)의 DNS 규칙 그대로입니다.
**ArgoCD가 설치된 것과 같은 클러스터에 배포할 때** 이 주소를 씁니다.

## 설치

### ① 전용 네임스페이스 만들기

```bash
kubectl create namespace argocd
```

> **네임스페이스**는 쿠버네티스 안의 **논리적 격리 공간**입니다.
> ArgoCD 관련 리소스를 다른 앱과 섞이지 않게 따로 두는 것입니다.
> → [kubernetes/06](../../kubernetes/06-오브젝트구조-metadata-namespace-label.md)

### ② 공식 매니페스트로 설치

```bash
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**이 한 줄로 API Server·Repo Server·Application Controller·Redis가 전부 설치됩니다.**

> **`--server-side`가 왜 필요한가?**
> ArgoCD의 매니페스트에는 **아주 큰 CRD 정의**가 들어 있습니다.
> 기본(클라이언트 사이드) 방식은 원본 전체를 어노테이션에 저장하는데, **크기 제한을 넘겨 실패**합니다.
> 서버 사이드 적용은 그 제한이 없습니다.

### ③ 상태 확인

```bash
kubectl get pods -n argocd
```

**모든 파드가 `Running`이 될 때까지 기다립니다.** 몇 분 걸릴 수 있습니다.

### ④ UI 접속

ArgoCD UI는 클러스터 안에 있으므로, 밖에서 접속하려면 **포트 포워딩**이 필요합니다.

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

> **`kubectl port-forward`** 는 **내 PC의 포트를 클러스터 안의 서비스로 이어주는 임시 터널**입니다.
> `8081:443` 은 **"내 PC의 8081로 오는 요청을 그 서비스의 443으로 보내라"** 는 뜻입니다.
>
> 이 명령은 **터미널이 열려 있는 동안만** 유지됩니다. 닫으면 연결도 끊깁니다.
> 실무에서는 Ingress로 정식 노출합니다. → [kubernetes/09](../../kubernetes/09-ingress.md)

### ⑤ 초기 비밀번호 확인

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

**읽는 법**: 시크릿에서 `password` 항목을 꺼내는데, 쿠버네티스 Secret은 값을 **Base64로 인코딩**해
저장하므로 `base64 -d`로 되돌립니다.

> **Base64는 암호가 아니라 인코딩입니다.** Secret이 안전한 이유는 인코딩 때문이 아니라
> **접근 권한(RBAC)으로 보호되기 때문**입니다. → [kubernetes/08](../../kubernetes/08-configmap과-secret.md)

`https://localhost:8081` 접속 → `admin` / 위에서 얻은 비밀번호로 로그인.

> **"주의 요함" 경고가 뜨는 것은 정상입니다.** 자체 서명 인증서를 쓰기 때문입니다.

## 상태 표시 읽는 법

ArgoCD UI에는 **두 종류의 상태**가 나란히 표시됩니다.

| 상태 | 뜻 | 값 |
|---|---|---|
| **Sync Status** | **Git과 클러스터가 일치하는가** | `Synced` / `OutOfSync` |
| **Health Status** | **배포된 것이 실제로 잘 돌고 있는가** | `Healthy` / `Progressing` / `Degraded` |

**둘은 별개입니다.**

```
   Synced + Degraded   → Git 대로 배포는 됐는데, 파드가 계속 죽고 있다
                          (이미지 태그가 틀렸거나, 앱이 시작에 실패하거나)
   OutOfSync + Healthy → 지금 돌아가는 건 멀쩡한데, Git 에 새 변경이 있다
```

**목표는 `Synced` + `Healthy`** 입니다.

## 정리(삭제)

```bash
# 1. 네임스페이스와 그 안의 리소스 전부 삭제
kubectl delete namespace argocd

# 2. 클러스터 전역 리소스(CRD) 삭제 — 완전 초기화 시
kubectl delete crd -l app.kubernetes.io/part-of=argocd
```

> **CRD는 네임스페이스에 속하지 않습니다(클러스터 전역).**
> 그래서 네임스페이스를 지워도 **남아 있습니다.** 완전히 지우려면 2번까지 해야 합니다.

## 참고 — WSL2에서 클러스터를 못 찾을 때

Windows에서 Docker Desktop의 쿠버네티스를 쓰면서 **WSL 안에서 `kubectl`을 쓰면** 이런 문제가 있습니다.

**증상**: `kubernetes.docker.internal` 주소를 찾지 못함 (DNS 조회 실패)

**원인**: 그 이름은 **Windows 쪽 hosts 파일에만** 등록되어 있고, WSL 리눅스는 모릅니다.

```bash
# 1. kubeconfig 를 WSL 로 복사
mkdir -p ~/.kube
cp /mnt/c/Users/<WINDOWS_USERNAME>/.kube/config ~/.kube/config

# 2. 권한 조정 (본인만 읽기 — 인증 정보이므로)
chmod 600 ~/.kube/config

# 3. WSL 의 hosts 에 이름을 직접 등록
sudo sh -c 'echo "127.0.0.1 kubernetes.docker.internal" >> /etc/hosts'

# 4. 확인
kubectl get nodes
```

```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   22s   v1.34.1
```

> **`/etc/hosts`** 는 **"이 이름은 이 IP다"를 수동으로 적어두는 파일**입니다.
> DNS 서버에 물어보기 전에 여기를 먼저 봅니다. `/mnt/c/`는 **WSL에서 Windows의 C 드라이브**를 가리킵니다.

---

[← GitOps](02-gitops.md) | [실습 1 — GitHub Actions와 ArgoCD →](04-실습-github-actions와-argocd.md)
