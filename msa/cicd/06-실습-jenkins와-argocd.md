> [← cicd 목차](README.md)

# 실습 2 — Jenkins와 ArgoCD

## 한 줄로

**CI는 Jenkins, CD는 ArgoCD.** 그리고 [실습 1](04-실습-github-actions와-argocd.md)과의 결정적 차이는 **애플리케이션 저장소와 매니페스트 저장소를 물리적으로 분리**한 것이다.

## 실습 1과 무엇이 다른가

| | **실습 1** | **실습 2 (지금)** |
|---|---|---|
| CI 도구 | GitHub Actions (SaaS) | **Jenkins** (직접 구축) |
| 저장소 | **하나** (코드 + 매니페스트) | **둘로 분리** ★ |
| 무한 루프 위험 | `paths-ignore`로 방어 필요 | **구조적으로 없음** |
| Webhook | GitHub 내부라 그냥 됨 | **터널링 필요** (로컬 Jenkins) |
| 자격 증명 | GitHub Secrets | **Jenkins Credentials** |

### 저장소를 왜 나누나

```
   [ 저장소 하나 ]
   argocd-project/
     ├── src/           ← 개발자가 매일 고침
     └── k8s/           ← CI 봇이 커밋함
                        → 이력이 섞이고, 무한 루프 위험이 있고,
                          개발자가 인프라 설정도 마음대로 바꿀 수 있다

   [ 저장소 둘 ]
   jenkins-project/     ← 애플리케이션 코드 (개발자 소유)
   jenkins-manifests/   ← 배포 설정 (인프라팀 소유, ArgoCD 가 감시)
```

**얻는 것**

| | 설명 |
|---|---|
| **무한 루프가 원천 차단** | CI가 커밋하는 곳(매니페스트 저장소)이 **CI를 트리거하는 곳이 아님** |
| **권한 분리** | 운영 설정 변경에 **별도 리뷰·승인**을 걸 수 있음 |
| **이력이 깨끗** | 코드 커밋과 배포 커밋이 섞이지 않음 |
| **재사용** | 여러 환경(dev/staging/prod)의 매니페스트를 한 저장소에 모을 수 있음 |

**대가**: 저장소가 둘이라 **관리 지점이 늘고**, CI가 **다른 저장소에 접근할 권한**을 가져야 합니다.

## 전체 흐름

```
[ 개발자 ]
   │ 1. 코드 Push (App Repo)
   ▼
[ GITHUB: APP REPO ]  (Spring Boot 소스)
   │ 2. Webhook 트리거 (Ngrok 경유)
   ▼
[ JENKINS — CI ]
   │
   ├─ 3. 빌드 & 테스트 (Java 21)
   ├─ 4. 도커 이미지 빌드 & Push  ──▶  [ Docker Hub ]
   │        (태그 = BUILD_NUMBER)
   ├─ 5. Manifest Repo 를 clone
   ├─ 6. values.yaml 의 이미지 태그 수정
   └─ 7. 커밋 & Push [bot]
   ▼
[ GITHUB: MANIFEST REPO ]  (Helm 차트)
   ▲
   │ 8. 변경 감지 (GitOps Pull)
   │
[ ARGOCD — CD ]
   │ 9. 자동 동기화 → 10. 새 이미지 받아 배포
   ▼
[ 쿠버네티스 클러스터 ]
```

**⑤~⑦이 실습 1과 다른 부분입니다.** 자기 저장소가 아니라 **남의 저장소를 clone해서 고칩니다.**

## 사전 준비

| # | 준비물 |
|---|---|
| 1 | Docker Hub 계정 + **PAT** |
| 2 | GitHub 계정 + **PAT** (Jenkins가 매니페스트 저장소에 Push해야 함) |
| 3 | 쿠버네티스 클러스터 + ArgoCD → [03](03-argocd.md) |

## 세 개의 디렉터리

```
jenkins-server/            ← Jenkins 인프라
├── Dockerfile
└── docker-compose.yml

jenkins-project/           ← 애플리케이션 (App Repo)
├── build.gradle
├── Dockerfile
├── Jenkinsfile            ★ 파이프라인 정의
├── argocd.yaml            ArgoCD Application (매니페스트 저장소를 가리킴)
└── src/main/java/...

jenkins-manifests/         ← 배포 설정 (Manifest Repo, 별도 저장소)
├── Chart.yaml
├── values.yaml            ★ Jenkins 가 자동 수정
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

---

## Step 1. Jenkins 서버 띄우기

### `jenkins-server/Dockerfile` — 왜 직접 이미지를 만드나

**공식 Jenkins 이미지에는 도커 CLI가 없습니다.** 그런데 우리 파이프라인은 이미지를 빌드해야 합니다.

```dockerfile
FROM jenkins/jenkins:lts-jdk21

USER root

# 도커 공식 GPG 키 등록 준비
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl gnupg \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && chmod a+r /etc/apt/keyrings/docker.gpg

# 도커 저장소 등록 + Docker CLI & Buildx 설치
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce-cli docker-buildx-plugin \
    && rm -rf /var/lib/apt/lists/*
```

**읽어야 할 점**

| | 설명 |
|---|---|
| **`lts-jdk21`** | LTS(장기 지원) 버전 + JDK 21 포함 |
| **`docker-ce-cli`만 설치** | 도커 **엔진이 아니라 명령어 도구만**. 엔진은 호스트 것을 빌려 씀 ([05 DooD](05-jenkins.md)) |
| **GPG 키 등록** | 패키지가 **위조되지 않았는지 검증**하기 위한 서명 키 |
| **`rm -rf /var/lib/apt/lists/*`** | 설치 후 **캐시 삭제 → 이미지 크기 감소** |

### `jenkins-server/docker-compose.yml`

```yaml
services:
  jenkins:
    build: .
    container_name: jenkins
    user: root                 # 호스트 도커 소켓 접근 권한 문제 회피 (실습용)
    ports:
      - "8085:8080"            # Jenkins Web UI
      - "50000:50000"          # Agent 접속 포트
    volumes:
      - ./jenkins_home:/var/jenkins_home          # ① 설정·데이터 보존
      - /var/run/docker.sock:/var/run/docker.sock # ② 호스트 도커 빌려쓰기
    restart: always
```

| 항목 | 왜 |
|---|---|
| **`8085:8080`** | Jenkins 기본 포트는 8080인데, **Spring Boot도 8080**이라 밖으로는 8085로 뺐음 |
| **① jenkins_home 볼륨** | **Jenkins의 모든 설정·잡·플러그인이 여기 저장됨.** 없으면 컨테이너를 지울 때 전부 사라짐 |
| **② 도커 소켓** | [05의 DooD](05-jenkins.md) |
| **`user: root`** | 소켓 파일 권한 때문. **실습용 편법**이며 운영에서는 권장되지 않음 |

> **Mac 사용자 참고**: Docker Desktop → Settings → Advanced에서
> **`Allow the default Docker socket to be used`** 가 체크되어 있어야 소켓 마운트가 동작합니다.

### 실행과 초기 설정

```bash
cd jenkins-server
docker compose up -d

# 최초 관리자 비밀번호 확인
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

`http://localhost:8085` 접속 →

```
① 위 비밀번호 입력
② [Install suggested plugins] 클릭 (설치 완료까지 대기)
③ 관리자 계정 생성 (admin / 원하는 비밀번호)
④ Jenkins URL: http://localhost:8085/ → Save and Finish
```

---

## Step 2. 파이프라인 정의 — `Jenkinsfile`

```groovy
pipeline {
  agent any

  environment {
    DOCKER_REPO       = "<도커허브ID>/jenkins-project"
    DOCKER_CRED_ID    = "DOCKERHUB_CREDENTIALS"
    MANIFEST_REPO_URL = "github.com/<깃허브ID>/jenkins-manifests.git"  // https:// 제외
    GIT_CRED_ID       = "GIT_PAT"
  }

  stages {
    stage('Checkout App') {
      steps { checkout scm }
    }

    stage('Build App') {
      steps {
        sh '''
          chmod +x gradlew
          ./gradlew clean bootJar -x test
        '''
      }
    }

    stage('Docker Buildx Multi-Arch Build & Push') { ... }   // 아래 설명

    stage('Update Manifest Repo (GitOps)') { ... }           // 아래 설명
  }

  post {
    always { cleanWs notFailBuild: true }
  }
}
```

### 이미지 빌드 단계

```groovy
withCredentials([usernamePassword(credentialsId: DOCKER_CRED_ID,
                                  usernameVariable: 'DOCKER_USER',
                                  passwordVariable: 'DOCKER_PASS')]) {
  sh """
    # 1. 로그인 (비밀번호를 표준 입력으로 — 프로세스 목록에 안 보이게)
    echo "\${DOCKER_PASS}" | docker login -u "\${DOCKER_USER}" --password-stdin

    # 2. Buildx 빌더 준비 (없으면 생성)
    docker buildx inspect multi-builder > /dev/null 2>&1 || \\
      docker buildx create --name multi-builder --driver docker-container --use
    docker buildx use multi-builder

    # 3. amd64 / arm64 동시 빌드 후 곧바로 레지스트리로 전송
    docker buildx build \\
      --platform linux/amd64,linux/arm64 \\
      -t ${DOCKER_REPO}:${BUILD_NUMBER} \\
      -t ${DOCKER_REPO}:latest \\
      --push .

    # 4. 인증 정보 정리
    docker logout
  """
}
```

**세 가지를 짚어야 합니다.**

**① `--push`를 반드시 써야 하는 이유**

```
   멀티 플랫폼 이미지는 로컬 도커 엔진에 저장할 수 없다
   (로컬 저장소는 아키텍처 하나만 담을 수 있는 구조)
   → 빌드하면서 곧바로 레지스트리로 보내야 한다
```

`docker build` 후 `docker push` 하는 평소 방식이 **여기서는 통하지 않습니다.**

**② `BUILD_NUMBER`**

Jenkins가 자동으로 제공하는 변수로, **빌드가 실행된 순번**입니다.
[실습 1의 `github.run_number`](04-실습-github-actions와-argocd.md)와 같은 역할입니다.

**③ `\${...}` 와 `${...}` 의 차이 — 헷갈리는 지점**

```groovy
sh """
  echo "\${DOCKER_PASS}"      // 백슬래시 O → 셸이 해석 (Jenkins 로그에 노출 안 됨)
  -t ${DOCKER_REPO}:${BUILD_NUMBER}   // 백슬래시 X → Groovy 가 먼저 치환
"""
```

| 표기 | 누가 해석하나 | 언제 씀 |
|---|---|---|
| `${VAR}` | **Groovy**가 문자열 만들 때 치환 | 파이프라인 변수 (`DOCKER_REPO` 등) |
| `\${VAR}` | **셸**이 실행할 때 치환 | **비밀값** — Groovy 단계에서 문자열에 박히지 않게 |

**비밀값에 백슬래시를 붙이는 이유**: Groovy가 치환해 버리면 **명령 문자열 자체에 비밀번호가 들어가**
로그나 오류 메시지로 샐 수 있습니다.

### 매니페스트 저장소 갱신 단계

```groovy
withCredentials([usernamePassword(credentialsId: GIT_CRED_ID,
                                  usernameVariable: 'GIT_USERNAME',
                                  passwordVariable: 'GIT_PASSWORD')]) {
  sh """
    git config --global user.email "jenkins-bot@users.noreply.github.com"
    git config --global user.name "jenkins-bot"

    rm -rf jenkins-manifests

    # PAT 를 URL 에 끼워 인증하며 clone
    git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${MANIFEST_REPO_URL} jenkins-manifests

    cd jenkins-manifests
    sed -i 's/tag: ".*"/tag: "${BUILD_NUMBER}"/g' values.yaml
    git add values.yaml

    if git diff --staged --quiet; then
      echo "No changes in values.yaml, skipping commit."
    else
      git commit -m "ci(gitops): update image tag to ${BUILD_NUMBER} [skip ci]"
      git pull --rebase origin main
      git push origin main
    fi
  """
}
```

**읽어야 할 네 가지**

| | 설명 |
|---|---|
| **`rm -rf` 먼저** | 이전 빌드에서 남은 디렉터리가 있으면 clone이 실패하므로 **매번 지우고 시작** |
| **`https://ID:PAT@주소`** | Git이 **비대화식으로 인증**하는 방법. `MANIFEST_REPO_URL`에 `https://`를 빼둔 이유가 이것 |
| **`git diff --staged --quiet`** | 변경이 없으면 **커밋을 건너뜀.** 안 하면 "빈 커밋" 오류로 파이프라인이 실패 |
| **`git pull --rebase` 후 push** | 그 사이 다른 커밋이 들어왔을 수 있으므로 **충돌 방지** |

> **`sed`, `[skip ci]`, `git pull --rebase`** 는 [실습 1](04-실습-github-actions와-argocd.md)에서 설명한 그대로입니다.

### `post { always { cleanWs() } }`

**빌드가 끝나면 작업 공간을 비웁니다.**

```
   안 지우면:  빌드마다 소스·빌드 산출물·clone 한 저장소가 쌓여 디스크가 참
   지우면:     다음 빌드가 항상 깨끗한 상태에서 시작 (재현성 ↑)
```

`notFailBuild: true` 는 **정리에 실패해도 빌드를 실패로 만들지 말라**는 뜻입니다.

---

## Step 3. Webhook 문제와 터널링

### 문제

```
   [ GitHub ]  ──"코드 바뀌었어!"──▶  [ localhost:8085 의 Jenkins ]
                                          ↑
                    인터넷에서 이 주소는 존재하지 않는다 ❌
```

**`localhost`와 `127.0.0.1`은 "내 컴퓨터"라는 뜻**입니다. GitHub 서버 입장에서
`localhost`는 **GitHub 자기 자신**입니다. 게다가 집·회사 네트워크는 **NAT과 방화벽** 뒤에 있어
외부에서 들어올 수 없습니다.

### 해법 두 가지

| 방법 | 설명 |
|---|---|
| **공인 IP를 가진 서버에 Jenkins 설치** | AWS EC2 등. **실무의 정답** |
| **터널링 서비스로 임시 노출** | ngrok 등. **실습에 적합** ★ |

### 기초 — 터널링이란

```
   [ GitHub ] ──▶ https://abcd.ngrok-free.dev  ← 인터넷에 존재하는 진짜 주소
                            │
                  ┌─────────▼──────────┐
                  │   ngrok 클라우드    │
                  └─────────┬──────────┘
                            │  내 PC 가 먼저 밖으로 연결해 둔 통로
                            ▼
                  [ 내 PC 의 localhost:8085 ]
```

**핵심은 "내가 먼저 밖으로 나가서 통로를 만들어 둔다"** 는 점입니다.
그러면 **방화벽을 열지 않고도** 외부 요청을 받을 수 있습니다.

> [02에서 본 GitOps의 Pull 방식](02-gitops.md)과 같은 발상입니다 — **나가는 연결만 쓴다.**

### ngrok 설치와 실행

```bash
# Ubuntu / WSL
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
sudo tar xvzf ngrok-v3-stable-linux-amd64.tgz -C /usr/local/bin
ngrok version
```

```bash
# Apple Silicon Mac
curl -O https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-darwin-arm64.zip
sudo unzip ngrok-v3-stable-darwin-arm64.zip -d /usr/local/bin
ngrok version
```

```bash
# 인증 토큰 등록 (최초 1회) — 대시보드에서 발급
ngrok config add-authtoken <토큰>

# Jenkins 포트를 외부에 노출
ngrok http 8085
```

**`https://랜덤문자.ngrok-free.dev`** 같은 주소가 생깁니다. 이 주소로 오는 요청이 `localhost:8085`로 전달됩니다.

> ⚠️ **무료 플랜은 재시작할 때마다 주소가 바뀝니다.** 바뀌면 **Webhook 설정도 다시 해야** 합니다.
>
> ⚠️ **인터넷 전체에 노출된다는 뜻이기도 합니다.** 실습이 끝나면 반드시 끄세요.

---

## Step 4. 저장소와 Jenkins 설정

### App Repo — `jenkins-project`

**Webhook 등록**: `Settings` → `Webhooks` → `Add webhook`

| 항목 | 값 |
|---|---|
| **Payload URL** | `https://[ngrok-주소]/github-webhook/` ← **마지막 슬래시까지** |
| Content type | `application/json` |
| Secret | 비움 |
| SSL verification | Enable |
| 트리거 이벤트 | `Just the push event.` |

> ⚠️ **`/github-webhook/` 의 마지막 슬래시를 빼면 동작하지 않습니다.** 가장 흔한 실수입니다.

```bash
git init
git remote add origin https://github.com/<깃허브ID>/jenkins-project.git
git add . && git commit -m 'initial' && git push -u origin main
```

### Manifest Repo — `jenkins-manifests`

**Webhook은 필요 없습니다.** 이 저장소는 **ArgoCD가 알아서 보러 옵니다.**

```bash
git init
git remote add origin https://github.com/<깃허브ID>/jenkins-manifests.git
git add . && git commit -m 'initial' && git push -u origin main
```

> **`values.yaml`의 `tag`가 [실습 1](04-실습-github-actions와-argocd.md)과 같은 역할**을 합니다.
> 차이는 **이 파일이 다른 저장소에 있다**는 것뿐입니다.

### Jenkins Credentials 등록

`⚙️ Jenkins 관리` → `Credentials` → 두 개 등록

| ID | Kind | Username | Password |
|---|---|---|---|
| **`DOCKERHUB_CREDENTIALS`** | Username with password | Docker Hub 아이디 | Docker Hub **PAT** |
| **`GIT_PAT`** | Username with password | GitHub 아이디 | GitHub **PAT** |

> ⚠️ **ID가 `Jenkinsfile`에 적힌 이름과 정확히 같아야 합니다.** 오타 하나면 파이프라인이 실패합니다.

**플러그인 설치**: `Jenkins 관리` → `Plugins` → `Available plugins` → **Docker Pipeline** 설치

### 파이프라인 Item 생성

`새로운 Item` → 이름 `jenkins-pipeline`, 종류 **Pipeline**

**Triggers 섹션**

```
✅ GitHub hook trigger for GITScm polling
```

> **읽는 법**: "GitHub에서 webhook 신호가 오면, 내가 설정한 Git 저장소를 확인하고
> 변경이 있으면 빌드를 시작하겠다."

**Pipeline 섹션**

| 항목 | 값 |
|---|---|
| Definition | **`Pipeline script from SCM`** |
| SCM | `Git` |
| Repository URL | `https://github.com/<깃허브ID>/jenkins-project.git` |
| Credentials | GitHub 자격 증명 선택 |
| Branch Specifier | `*/main` |
| **Script Path** | **`Jenkinsfile`** |

> **`Pipeline script from SCM`의 뜻**: 파이프라인 내용을 **Jenkins UI에 저장하지 않고,
> Git 저장소의 `Jenkinsfile`을 읽어서 쓰겠다**는 것입니다.
> → [05의 Pipeline as Code](05-jenkins.md)

---

## Step 5. 실행과 검증

### 1) 최초는 수동 실행

**Webhook은 "이벤트가 생겼을 때"만 동작**합니다. 이미 Push된 코드는 자동으로 빌드되지 않습니다.

```
jenkins-pipeline → [지금 빌드] 클릭
```

**실패하면**: 빌드 번호(`#1`) → **Console Output**에서 로그 확인.
원인을 고쳐 Push하면 **이번에는 Webhook으로 자동 빌드(`#2`)** 됩니다.

### 2) 결과 확인

| 확인할 곳 | 무엇을 |
|---|---|
| Jenkins UI | 모든 stage가 초록색인가 |
| Docker Hub | `jenkins-project` 이미지가 올라왔는가 |
| **Manifest Repo** | `values.yaml`의 `tag`가 **자동으로 바뀌었는가** ★ |

### 3) ArgoCD 연결

```bash
kubectl apply -f argocd.yaml
kubectl port-forward svc/argocd-server -n argocd 8081:443

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

`argocd.yaml`이 **매니페스트 저장소를 가리키는 것**이 실습 1과 다른 점입니다.

```yaml
source:
  repoURL: 'https://github.com/<깃허브ID>/jenkins-manifests.git'   # ← Manifest Repo
  targetRevision: main
  path: .          # 저장소 루트에 Chart.yaml 이 있으므로 '.'
```

`https://localhost:8081` 로그인 → **`Synced` + `Healthy`** 확인.

### 4) 전체 고리 돌려보기

```
① jenkins-project 의 JenkinsController 반환값을 (v1) → (v2) 로 수정
② App Repo 에 Push
③ Webhook → Jenkins 자동 빌드 시작
④ 이미지 tag: 2 를 Docker Hub 에 Push
⑤ Manifest Repo 의 values.yaml 을 tag: "2" 로 커밋
⑥ ArgoCD 가 감지 (자동 또는 Refresh 버튼)
⑦ 롤링 업데이트 → 브라우저에서 (v2) 확인
```

**저장소 두 개, 도구 세 개(Jenkins·Docker Hub·ArgoCD)가 사람 손 없이 이어지는 것**이
이 실습의 결말입니다.

---

## Step 6. 정리

```bash
# 1. 터널링 · 포트포워딩 종료
pkill -f ngrok || true
pkill -f "kubectl port-forward" || true

# 2. 배포된 애플리케이션 삭제
kubectl delete deployment jenkins-project --ignore-not-found=true
kubectl delete service jenkins-project --ignore-not-found=true

# 3. ArgoCD 삭제
kubectl delete namespace argocd --ignore-not-found=true
kubectl delete crd -l app.kubernetes.io/part-of=argocd --ignore-not-found=true

# 4. Jenkins 및 도커 리소스 종료
docker compose down -v
docker rm -f buildx_buildkit_multi-builder0

# 5. 확인
docker ps
kubectl get pods -A
```

> **`docker compose down -v`의 `-v`는 볼륨까지 삭제**합니다.
> `jenkins_home`이 지워지므로 **Jenkins 설정이 전부 사라집니다.** 유지하려면 `-v`를 빼세요.
>
> **`buildx_buildkit_multi-builder0`** 은 Buildx가 만든 **빌더 전용 컨테이너**입니다.
> `docker compose down`으로는 안 지워지므로 따로 지웁니다.

---

## 자주 만나는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| Webhook이 안 옴 | Payload URL 마지막 **슬래시 누락** | `/github-webhook/` |
| 같음 | ngrok 주소가 **바뀜** | Webhook 재설정 |
| 같음 | Triggers 체크 안 함 | `GitHub hook trigger...` 확인 |
| `docker: command not found` | Jenkins 이미지에 CLI 미설치 | 커스텀 Dockerfile 확인 |
| `permission denied ... docker.sock` | 소켓 권한 | `user: root` 또는 Mac 설정 확인 |
| Credentials를 못 찾음 | **ID 오타** | Jenkinsfile의 ID와 대조 |
| Manifest Repo에 Push 실패 | GitHub PAT 권한 부족 | `repo` 권한 포함 재발급 |
| `--push` 없이 멀티 빌드 실패 | 멀티 플랫폼은 로컬 저장 불가 | `--push` 사용 |
| ArgoCD가 Manifest 변경을 못 봄 | `repoURL`이 App Repo를 가리킴 | **Manifest Repo**로 수정 |
| 빈 커밋 오류 | 변경이 없는데 커밋 시도 | `git diff --staged --quiet` 분기 |

## 두 실습 비교 — 언제 무엇을 쓰나

| | **실습 1 (GitHub Actions)** | **실습 2 (Jenkins)** |
|---|---|---|
| 시작 난이도 | **아주 쉬움** | 서버 구축 필요 |
| 운영 부담 | **없음** (SaaS) | Jenkins를 직접 관리 |
| 폐쇄망 | 어려움 | **가능** |
| 저장소 구조 | 하나 (간단하지만 루프 위험) | **분리 (권장)** |
| 적합한 상황 | 소규모 팀, 오픈소스, 빠른 시작 | 규제 산업, 사내 인프라, 세밀한 통제 |

> **저장소 분리는 CI 도구와 무관합니다.** GitHub Actions로도 저장소를 나눌 수 있고,
> Jenkins로도 하나만 쓸 수 있습니다. **두 축을 따로 판단하세요.**

---

[← Jenkins](05-jenkins.md)
