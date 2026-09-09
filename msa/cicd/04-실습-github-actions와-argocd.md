> [← cicd 목차](README.md)

# 실습 1 — GitHub Actions와 ArgoCD

## 한 줄로

**CI는 GitHub Actions가, CD는 ArgoCD가** 맡는다. 저장소 **하나에** 애플리케이션 코드와 매니페스트를 함께 두는 가장 단순한 GitOps 구성.

## 전체 흐름

```
[ 개발자 ]
   │ 코드 Push
   ▼
[ GitHub 저장소 ]  (애플리케이션 코드 + Helm 차트)
   │ 트리거 (push: main)
   ▼
[ GitHub Actions — CI ]
   │
   ├─ 1. 빌드 & 테스트 (Java 21)
   ├─ 2. 도커 이미지 빌드 & Push  ──▶  [ Docker Hub ]
   │        (태그 = 빌드 번호)
   ├─ 3. values.yaml 의 이미지 태그를 새 번호로 수정
   └─ 4. 그 변경을 Git 에 커밋 & Push  [bot]
   ▼
[ GitHub 저장소 ]
   ▲
   │ 5. 변경 감지 (GitOps Pull)
   │
[ ArgoCD — CD ]
   │ 6. 자동 동기화 & 롤링 업데이트
   ▼
[ 쿠버네티스 클러스터 ]
```

> **③④가 이 실습의 핵심입니다.**
> CI는 **클러스터를 전혀 건드리지 않습니다.** 그냥 **Git에 커밋만** 합니다.
> 그 다음은 ArgoCD가 알아서 합니다. → [02 GitOps](02-gitops.md)

## 사전 준비

| # | 준비물 | 비고 |
|---|---|---|
| 1 | Docker Hub 계정 + **PAT** | 이미지를 올릴 곳 |
| 2 | GitHub 저장소 | 코드와 매니페스트를 둘 곳 |
| 3 | 쿠버네티스 클러스터 + **ArgoCD 설치 완료** | → [03](03-argocd.md) |

> **PAT = Personal Access Token (개인 액세스 토큰).**
> 비밀번호 대신 쓰는 **용도가 제한된 열쇠**입니다.
>
> | | 비밀번호 | PAT |
> |---|---|---|
> | 권한 | **계정 전부** | **필요한 것만** (예: 이미지 push만) |
> | 폐기 | 바꾸면 모든 곳이 끊김 | **그 토큰만 폐기** 가능 |
> | 만료 | 없음 | **기한 설정 가능** |
>
> **자동화에는 반드시 PAT를 씁니다.** → [container/05 레지스트리 인증](../../container/05-레지스트리-인증.md)

## 프로젝트 구조

```
argocd-project
├── build.gradle
├── Dockerfile                   # 이미지 빌드용
├── argocd.yaml                  # ArgoCD Application 리소스
│
├── .github/workflows/
│   └── ci-pipeline.yml          # GitHub Actions 워크플로
│
├── k8s/helm/                    # Helm 차트
│   ├── Chart.yaml
│   ├── values.yaml              # ★ CI 가 자동으로 수정하는 파일
│   └── templates/
│       ├── deployment.yaml
│       └── service.yaml
│
└── src/main/java/com/example/argocd/
    ├── ArgocdProjectApplication.java
    └── controller/ArgocdController.java
```

---

## Step 1. 애플리케이션과 이미지

### 확인용 컨트롤러

```java
@RestController
public class ArgocdController {
  @GetMapping("/")
  public String getHost() {
    String hostName = "Unknown";
    try {
      hostName = InetAddress.getLocalHost().getHostName();
    } catch (UnknownHostException e) { ... }
    // 배포 확인을 위해 v1 → v2 로 바꿔가며 실습
    return "CI/CD Pipeline (v1) - Host: " + hostName;
  }
}
```

**두 가지를 확인하려고 이렇게 만들었습니다.**

| 반환값 | 무엇을 확인 |
|---|---|
| `(v1)` / `(v2)` | **새 버전이 실제로 배포됐는지** |
| `Host: ...` | **파드 이름** — 요청이 여러 파드에 분산되는지 |

> 컨테이너 안에서 호스트명은 **파드 이름**이 됩니다. 새로고침할 때마다 바뀌면
> **replica 2개에 로드밸런싱되고 있다**는 뜻입니다.

### Dockerfile — 멀티 스테이지 빌드

```dockerfile
# ===== STAGE 1: 빌드 =====
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /demo
COPY build.gradle settings.gradle /demo/
COPY gradle /demo/gradle
COPY gradlew /demo/
RUN chmod +x ./gradlew
COPY src /demo/src
RUN ./gradlew clean bootJar -x test --no-daemon

# ===== STAGE 2: 실행 =====
FROM eclipse-temurin:21-jre
WORKDIR /demo
COPY --from=builder /demo/build/libs/*.jar demo.jar
EXPOSE 8080
CMD ["java", "-jar", "demo.jar"]
```

### 기초 — 멀티 스테이지 빌드가 뭔가

**문제**: 자바를 **빌드**하려면 JDK(컴파일러 포함)가 필요하지만,
**실행**할 때는 JRE(실행 환경)만 있으면 됩니다.

```
   한 단계로 만들면:  JDK + Gradle + 소스코드 + 빌드 캐시 + 결과물  →  이미지 700MB+
   두 단계로 나누면:  JRE + JAR 파일만                              →  이미지 250MB
```

**`--from=builder` 가 핵심입니다.**

```
   [ Stage 1: builder ]  JDK 로 빌드 → JAR 생성
            │
            │  COPY --from=builder   ← 결과물만 꺼내온다
            ▼
   [ Stage 2 ]  JRE 위에 JAR 만 얹는다 (Stage 1 은 버려짐)
```

**작은 이미지가 좋은 이유**

| | 이유 |
|---|---|
| **배포가 빠르다** | 내려받을 용량이 적음 |
| **보안이 낫다** | 컴파일러·빌드 도구가 없으니 **공격 표면이 작음** |
| **비용이 싸다** | 레지스트리 저장 용량 |

> **`COPY` 순서에도 이유가 있습니다.** 소스(`src`)를 **마지막에** 복사합니다.
> 도커는 **레이어 단위로 캐시**하므로, 소스만 바뀌었을 때 **의존성 다운로드 단계는 캐시를 재사용**합니다.
> 순서를 바꾸면 매번 처음부터 받습니다. → [container/06 이미지 태그 전략](../../container/06-이미지-태그-전략.md)

---

## Step 2. Helm 차트

### `Chart.yaml` — 이름표

```yaml
apiVersion: v2
name: argocd-project
description: Kubernetes CI/CD pipeline with GitHub Actions and ArgoCD
type: application
version: 0.1.0
appVersion: "1.0.0"
```

**이 파일이 있으면 ArgoCD가 "Helm 차트구나" 하고 인식**해서, 알아서 렌더링한 뒤 적용합니다.

```
① ArgoCD 가 Git 에서 Chart.yaml 을 발견
② 스스로 템플릿을 렌더링해서 최종 YAML 생성
③ 클러스터에 apply
```

### `values.yaml` — CI가 고치는 파일

```yaml
replicaCount: 2

image:
  repository: <도커허브ID>/argocd-project
  pullPolicy: IfNotPresent
  tag: "1"                 # ★ CI 가 빌드 번호로 자동 수정하는 부분

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080         # Spring Boot 포트
```

**`tag` 한 줄이 이 파이프라인의 연결 고리입니다.**
CI가 이 줄을 바꿔 커밋하면 → ArgoCD가 그 변경을 보고 → 새 이미지로 배포합니다.

> **`pullPolicy: IfNotPresent`** 는 **"이미 노드에 있으면 다시 안 받는다"** 는 뜻입니다.
> **그래서 태그를 매번 바꾸는 것이 중요합니다.**
> `latest` 같은 고정 태그를 쓰면 **새 이미지를 밀어 올려도 노드가 옛 이미지를 그대로 씁니다.**

### `templates/deployment.yaml`

```yaml
spec:
  replicas: {{ .Values.replicaCount }}                        # 2
  ...
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
```

**`livenessProbe`가 왜 여기 있나**

> **프로브(Probe)** 는 쿠버네티스가 **컨테이너 상태를 확인하는 방법**입니다.
> `livenessProbe`가 실패하면 **쿠버네티스가 컨테이너를 죽이고 다시 띄웁니다.**
>
> `/actuator/health`는 Spring Boot Actuator가 제공하는 상태 확인 주소입니다.
> → [observation/02](../observation/02-메트릭.md) · [kubernetes/11 프로브](../../kubernetes/11-노드-rollout-프로브.md)
>
> **`initialDelaySeconds: 30`** — 앱이 뜨는 데 시간이 걸리므로 **30초는 봐 준다**는 뜻입니다.
> 이게 너무 짧으면 **시작 중인 앱을 계속 죽여서 무한 재시작**에 빠집니다.

### `templates/service.yaml`

```yaml
spec:
  type: {{ .Values.service.type }}          # LoadBalancer
  ports:
    - port: {{ .Values.service.port }}        # 80  (밖에서 접속할 포트)
      targetPort: {{ .Values.service.targetPort }}   # 8080 (컨테이너 포트)
  selector:
    app: {{ .Chart.Name }}                  # ★ Deployment 의 label 과 일치해야 함
```

> **`selector`가 Deployment의 label과 다르면 트래픽이 아무 데도 안 갑니다.**
> Service는 **label로 파드를 찾기** 때문입니다. 여기서는 둘 다 `.Chart.Name`을 써서 자동으로 맞춥니다.
> → [kubernetes/05 워크로드](../../kubernetes/05-워크로드-deployment-replicaset-service.md)

---

## Step 3. GitHub Actions 워크플로

### 기초 — GitHub Actions란

**GitHub 저장소에 이벤트가 생기면 자동으로 스크립트를 실행해 주는 서비스**입니다.

| 용어 | 뜻 |
|---|---|
| **Workflow** | `.github/workflows/*.yml` 파일 하나 = 자동화 흐름 하나 |
| **Job** | 워크플로 안의 **작업 묶음**. Job끼리는 기본적으로 **병렬** 실행 |
| **Step** | Job 안의 **개별 명령** |
| **Runner** | 실제로 실행되는 **가상 머신** (`ubuntu-latest` 등) |
| **Action** | 남이 만들어 둔 **재사용 가능한 단계** (`uses:` 로 가져다 씀) |

**서버를 직접 운영하지 않아도 된다**는 게 가장 큰 장점입니다(SaaS형).

### 트리거

```yaml
on:
  push:
    branches: [ "main" ]
    paths-ignore:
      - 'k8s/**'          # ★ 무한 빌드 루프 방지
```

**`paths-ignore`가 왜 결정적으로 중요한가**

```
   ① 코드 Push → CI 실행
   ② CI 가 values.yaml 을 고쳐서 Git 에 커밋
   ③ 그 커밋이 또 push 이벤트를 발생 → CI 실행
   ④ CI 가 또 values.yaml 을 고쳐서 커밋
   ⑤ ... 무한 반복 💀 (GitHub Actions 사용 시간이 소진됨)
```

**저장소 하나에 코드와 매니페스트를 같이 두면 반드시 생기는 문제**입니다.
`k8s/` 아래 변경은 무시하도록 해서 고리를 끊습니다.

> 커밋 메시지의 **`[skip ci]`** 도 같은 목적의 안전장치입니다(뒤에 나옵니다).
> **둘 다 걸어두는 게 안전합니다.**

### Job 1 — 빌드와 이미지 Push

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read                        # 코드 읽기만

    steps:
      - name: Checkout code
        uses: actions/checkout@v4           # 저장소 코드를 러너로 가져옴

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'                   # 의존성 캐싱 → 빌드 시간 단축

      - name: Build with Gradle
        run: |
          chmod +x gradlew
          ./gradlew clean build

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64       # 두 아키텍처 동시 빌드
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/argocd-project:${{ github.run_number }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**이해해야 할 네 가지**

**① `secrets.XXX` — 비밀값 주입**

```
   GitHub 저장소 설정에 미리 등록해 둔 값을 여기서 꺼내 쓴다
   → 로그에도 ***** 로 가려져 출력된다
```

**절대 워크플로 파일에 비밀번호를 직접 쓰지 마세요.** 저장소가 공개되면 그대로 노출됩니다.

**② `github.run_number` — 이미지 태그**

**워크플로가 실행된 순번**(1, 2, 3…)입니다. 이걸 이미지 태그로 씁니다.

```
   myid/argocd-project:1
   myid/argocd-project:2   ← 실행할 때마다 자동으로 하나씩 증가
```

> **`latest`를 안 쓰는 이유**: **어느 코드가 배포된 건지 알 수 없고, 롤백이 불가능**합니다.
> **모든 빌드에 고유한 태그**를 붙이는 것이 원칙입니다. → [container/06](../../container/06-이미지-태그-전략.md)

**③ QEMU + Buildx — 여러 아키텍처 동시 빌드**

| 도구 | 하는 일 |
|---|---|
| **QEMU** | 다른 CPU 아키텍처를 **에뮬레이션** (x86 머신에서 ARM 빌드) |
| **Buildx** | 도커의 **확장 빌드 기능**. 멀티 플랫폼·고급 캐시 지원 |

**왜 필요한가**: 개발자는 **Apple Silicon Mac(ARM)**, 서버는 **x86(amd64)** 인 경우가 흔합니다.
아키텍처가 다르면 **이미지가 아예 실행되지 않습니다**(`exec format error`).

**④ `cache-from/to: type=gha` — 캐시**

GitHub Actions의 캐시 저장소에 **도커 빌드 레이어를 보관**해 다음 빌드를 빠르게 합니다.
러너는 **매번 새 가상 머신**이라, 캐시를 밖에 두지 않으면 항상 처음부터 빌드합니다.

### Job 2 — 매니페스트 갱신 (GitOps 연결점)

```yaml
  deploy:
    needs: build                    # build 가 성공해야만 실행
    runs-on: ubuntu-latest
    permissions:
      contents: write               # ★ 커밋·푸시를 해야 하므로 쓰기 권한

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          ref: main

      - name: Update Helm Values
        run: |
          sed -i 's/tag: ".*"/tag: "${{ github.run_number }}"/g' k8s/helm/values.yaml
          cat k8s/helm/values.yaml

      - name: Commit and Push Manifest changes
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/helm/values.yaml

          # 변경이 있을 때만 커밋 (빈 커밋 에러 방지)
          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            git commit -m "ci(gitops): update image tag to ${{ github.run_number }} [skip ci]"
            git push origin main
          fi
```

**이 Job이 "CI와 CD를 잇는 다리"입니다.** 클러스터를 건드리지 않고 **Git에 커밋만** 합니다.

### 기초 — `sed` 명령어 읽기

```bash
sed -i 's/tag: ".*"/tag: "5"/g' k8s/helm/values.yaml
```

**`sed` = Stream EDitor.** 텍스트를 **찾아서 바꾸는** 리눅스 도구입니다.

| 조각 | 뜻 |
|---|---|
| `-i` | **In-place** — 파일을 **직접 수정**해라 (없으면 화면에만 출력) |
| `s/A/B/g` | **s**ubstitute — A를 B로 바꿔라. `g`는 **한 줄에 여러 개 있어도 전부** |
| `.*` | 정규식 — **아무 문자가 0개 이상** |
| `/` (구분자) | `s`와 값들을 나누는 기호. **바꿀 내용에 `/`가 들어가면** `s\|A\|B\|g` 처럼 다른 기호를 씀 |

```
   tag: ".*"   →  tag: 다음의 따옴표 안 아무 내용이나 매칭
   tag: "5"    →  그것을 이걸로 교체
```

> **도커 이미지 주소(`myid/myapp`)에는 `/`가 들어갑니다.** 그럴 때 구분자를 `/`로 쓰면
> `sed`가 헷갈리므로 **`|`나 `#`** 를 씁니다.

### 기초 — `[skip ci]` 규약

커밋 메시지에 **`[skip ci]`** 또는 **`[ci skip]`** 이 들어 있으면,
**대부분의 CI 도구가 그 커밋에 대해 파이프라인을 실행하지 않습니다.**

```
   git commit -m "ci(gitops): update image tag to 5 [skip ci]"
                                                    ↑ 이 커밋으로는 CI 를 돌리지 마라
```

GitHub Actions, GitLab CI, Jenkins, Bitbucket 등이 **공통으로 인식하는 관례**입니다.
**무한 빌드 루프를 막는 두 번째 안전장치**입니다.

### `permissions`가 Job마다 다른 이유

```yaml
build:   permissions: { contents: read }    # 코드만 읽으면 됨
deploy:  permissions: { contents: write }   # 커밋·푸시를 해야 함
```

**최소 권한 원칙(Principle of Least Privilege)** 입니다.
**필요한 만큼만 주면**, 워크플로에 문제가 생겨도 **피해 범위가 작습니다.**

> **`secrets.GITHUB_TOKEN`** 은 GitHub가 **워크플로 실행마다 자동으로 만들어 주는 임시 토큰**입니다.
> 내가 등록할 필요가 없고, **워크플로가 끝나면 만료**됩니다.

---

## Step 4. 실행과 검증

### 1) GitHub 저장소 설정

```bash
git init
git remote add origin https://github.com/<USER>/argocd-project.git
```

**Secrets 등록**

`Settings` → `Secrets and variables` → `Actions` → `New repository secret`

| 이름 | 값 |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub 아이디 |
| `DOCKERHUB_PASSWORD` | Docker Hub **PAT** |

**Workflow 권한 확인**

`Settings` → `Actions` → `General` → `Workflow permissions`
→ **`Read and write permissions`** 선택

> ⚠️ **기본값은 읽기 전용입니다.** 이걸 안 바꾸면
> **deploy Job의 `git push`가 403으로 실패**합니다. 가장 흔한 첫 실패 원인입니다.

### 2) 코드 Push

```bash
git add .
git commit -m 'feat: initial commit'
git push origin main
```

### 3) CI 결과 확인

| 확인할 곳 | 무엇을 |
|---|---|
| GitHub `Actions` 탭 | 파이프라인이 초록불로 끝났는가 |
| Docker Hub | `argocd-project:1` 이미지가 올라왔는가 |
| `k8s/helm/values.yaml` | `tag: "1"` 로 **자동 변경**되었는가 (봇 커밋 확인) |

**세 번째가 이 실습의 핵심**입니다. **기계가 스스로 Git에 커밋했다는 것**이 GitOps의 시작점입니다.

### 4) ArgoCD 연결

```bash
# Application 리소스 등록 (최초 1회만)
kubectl apply -f argocd.yaml

# UI 접속용 터널
kubectl port-forward svc/argocd-server -n argocd 8081:443

# 초기 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

`https://localhost:8081` → `admin` / 위 비밀번호

**`Synced` + `Healthy`** 가 뜨면 성공입니다. → [03의 상태 읽는 법](03-argocd.md)

### 5) 변경 배포 — 전체 고리 돌려보기

**① 코드 수정**: `ArgocdController`의 반환 문자열을 `(v1)` → `(v2)` 로

**② Push**

```bash
# 봇이 원격에 커밋을 만들어 뒀으므로 먼저 가져와야 한다
git pull --rebase origin main

git add .
git commit -m 'feat: v2'
git push origin main
```

> **`--rebase`가 왜 필요한가**
>
> ```
>   원격:  A ─ B ─ [봇 커밋]        ← CI 가 values.yaml 을 고친 커밋
>   로컬:  A ─ B ─ [내 커밋]        ← 봇 커밋을 모르는 상태
>          → 그냥 push 하면 거부됨 (rejected)
> ```
>
> **`git pull --rebase`** 는 **내 커밋을 잠시 떼어 두고, 원격 최신을 받은 뒤, 그 위에 다시 얹습니다.**
> 이력이 **한 줄로 깔끔하게** 유지됩니다. (그냥 `pull`은 병합 커밋이 하나 더 생깁니다.)
>
> **CI가 저장소에 커밋을 하는 구조에서는 이 습관이 필수**가 됩니다.

**③ 자동으로 벌어지는 일**

```
   GitHub Actions 실행  →  이미지 tag: 2 Push  →  values.yaml 을 tag: "2" 로 커밋
                                                        │
   ArgoCD 가 감지 (기본 3분 주기 또는 Refresh 버튼)  ◀────┘
        │
        ▼
   새 파드 생성 (ContainerCreating) → 기존 파드 종료 (Terminating)
```

**ArgoCD UI에서 이 과정을 시각적으로 볼 수 있습니다.**

> **롤링 업데이트(Rolling Update)** 는 **파드를 한꺼번에 바꾸지 않고 하나씩 교체**하는 방식입니다.
> 새 파드가 준비되면 옛 파드를 내리므로 **서비스가 끊기지 않습니다.**
> → [kubernetes/11](../../kubernetes/11-노드-rollout-프로브.md)

### 6) 정리

```bash
# ArgoCD UI 에서 앱 카드 → Delete

# 포트 포워딩 종료 (해당 터미널에서 Ctrl+C)

# ArgoCD 삭제
kubectl delete namespace argocd
kubectl delete crd -l app.kubernetes.io/part-of=argocd

# 로컬 이미지 정리
docker rmi <도커허브ID>/argocd-project:2
docker image prune -f
```

---

## 자주 만나는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| deploy Job이 **403**으로 실패 | Workflow 권한이 읽기 전용 | `Read and write permissions` 선택 |
| Docker Hub 로그인 실패 | Secret 이름 오타 / PAT 만료 | Secrets 재확인 |
| **CI가 무한 반복** | `paths-ignore` 또는 `[skip ci]` 누락 | 둘 다 적용 |
| ArgoCD가 `OutOfSync`인데 안 바뀜 | 자동 동기화 미설정 | `syncPolicy.automated` 확인 |
| 이미지를 못 받음 (`ImagePullBackOff`) | 태그 오타 / 이미지가 아직 push 안 됨 | Docker Hub에서 태그 확인 |
| 파드가 계속 재시작 | `livenessProbe` 실패 | `initialDelaySeconds` 늘리기, Actuator 노출 확인 |
| **새 이미지를 올렸는데 안 바뀜** | 태그를 고정으로 씀 (`latest`) | **빌드마다 고유 태그** |
| `git push` 거부 | 봇 커밋을 안 받아옴 | `git pull --rebase origin main` |

## 이 구성의 한계

| 한계 | 설명 |
|---|---|
| **저장소가 하나** | CI가 자기 저장소에 커밋 → **무한 루프 위험**을 계속 관리해야 함 |
| **권한이 섞임** | 애플리케이션 개발자가 **인프라 매니페스트도 수정 가능** |
| **이력이 섞임** | 커밋 로그에 코드 변경과 봇 커밋이 뒤섞임 |

**실무에서는 저장소를 분리하는 경우가 많습니다.** → [실습 2](06-실습-jenkins와-argocd.md)

---

[← ArgoCD](03-argocd.md) | [Jenkins →](05-jenkins.md)
