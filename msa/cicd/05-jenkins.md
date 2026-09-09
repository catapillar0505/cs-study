> [← cicd 목차](README.md)

# Jenkins

## 한 줄로

**직접 설치해서 쓰는 CI/CD 자동화 서버**다. GitHub Actions 같은 SaaS와 달리 **내 서버에 두고 100% 통제**하며, 플러그인으로 거의 모든 것에 연결한다.

## 무엇인가

- Hudson 프로젝트에서 갈라져 나온 **오픈소스 CI/CD 자동화 서버**
- 소스 통합·빌드·테스트·배포 파이프라인 전체를 자동화
- **온프레미스(직접 구축)** 환경에 설치해 **보안과 제어권을 완전히 확보**할 수 있음

> **온프레미스(On-Premise)** 는 "구내에"라는 뜻으로, **클라우드가 아니라 우리가 직접 관리하는 서버**를 말합니다.
> 반대말이 **SaaS**(서비스형 소프트웨어) — GitHub Actions처럼 **남이 운영해 주는 것**입니다.

## GitHub Actions와 비교

| | **GitHub Actions** | **Jenkins** |
|---|---|---|
| 운영 주체 | **GitHub이 운영** (SaaS) | **내가 설치·운영** |
| 초기 설정 | 파일 하나 넣으면 끝 | **서버 구축·플러그인 설치 필요** |
| 비용 | 사용량 기반 (무료 한도 있음) | **소프트웨어는 무료**, 서버 비용은 내 몫 |
| 폐쇄망 | **어려움** (인터넷 필요) | **가능** ★ |
| 확장성 | GitHub이 제공하는 러너 | **플러그인 1,800개 이상** |
| 통제권 | 제한적 | **전부 내 손에** |

**언제 Jenkins를 고르나**

```
✅ 인터넷이 안 되는 폐쇄망(금융·공공)
✅ 빌드 서버의 사양·환경을 완전히 통제해야 할 때
✅ 특수한 도구·레거시 시스템과 연동해야 할 때
✅ 이미 사내에 Jenkins 자산이 많을 때

❌ 팀이 작고 인프라를 관리할 여력이 없을 때  → SaaS 가 낫다
```

## 핵심 특징

| 특징 | 설명 |
|---|---|
| **오픈소스** | 라이선스 비용 없이 설치·커스터마이징 가능 |
| **플러그인 생태계** | Git·Docker·Kubernetes·Slack 등 **1,800개 이상** |
| **Pipeline as Code** | 파이프라인 전체를 **`Jenkinsfile` 코드로 정의**해 Git으로 버전 관리 |
| **분산 빌드** | **Controller + Agent** 구조로 여러 노드에서 병렬 빌드 |

## Controller / Agent 구조

```
            ┌───────────────────────────────────┐
            │        JENKINS CONTROLLER         │
            │   (UI, 작업 스케줄링, 파이프라인)    │
            └────────┬─────────────────┬────────┘
                     │ 작업 배정         │ 작업 배정
          ┌──────────▼───────┐ ┌───────▼──────────┐
          │ JENKINS AGENT 01 │ │ JENKINS AGENT 02 │
          │  (빌드 & 테스트)   │ │  (도커 빌드)      │
          └──────────────────┘ └──────────────────┘
```

| | 역할 |
|---|---|
| **Controller** | **두뇌.** UI 제공, 언제 무엇을 실행할지 결정, 결과 수집 |
| **Agent** | **손발.** 실제로 빌드·테스트를 수행하는 별도 머신(또는 컨테이너) |

**왜 나누나?**

```
   ❌ Controller 에서 전부 빌드하면
      → 빌드가 몰리면 UI 까지 느려짐
      → 빌드 환경(JDK 버전 등)을 바꾸려면 Controller 를 건드려야 함

   ✅ Agent 로 분리하면
      → 동시에 여러 빌드를 병렬 처리
      → 프로젝트마다 다른 환경의 Agent 사용 (Java 17 용, Java 21 용, Node 용…)
      → Agent 가 죽어도 Controller 는 멀쩡
```

> **소규모나 실습에서는 Controller 하나로도 충분**합니다.
> `agent any` 라고 쓰면 **"쓸 수 있는 아무 곳에서나 실행"** 이라는 뜻이고, Agent가 없으면 Controller가 직접 합니다.

## Jenkinsfile — Pipeline as Code

**저장소 루트의 `Jenkinsfile`을 읽어 파이프라인을 실행합니다.**

### 왜 파일로 관리하나

```
   ❌ Jenkins UI 에서 클릭으로 설정
      → 설정이 Jenkins 서버 안에만 있음
      → 서버가 죽으면 설정도 사라짐
      → 누가 언제 왜 바꿨는지 모름
      → 코드 리뷰 불가

   ✅ Jenkinsfile 로 저장소에 두면
      → 코드와 파이프라인이 같이 버전 관리됨
      → 브랜치마다 다른 파이프라인 가능
      → PR 로 리뷰 가능
```

**[GitOps](02-gitops.md)와 같은 발상입니다** — **"설정을 Git에 둔다."**

### Groovy DSL

Jenkinsfile은 **Groovy**라는 언어의 문법을 씁니다.

> **Groovy**는 JVM 위에서 도는 스크립트 언어입니다. 자바와 문법이 비슷합니다.
> **DSL(Domain Specific Language)** 은 **특정 목적에 특화된 문법**이라는 뜻입니다.
> Groovy를 몰라도 **정해진 틀만 채우면** 됩니다.

**두 가지 작성 방식이 있습니다.**

| | **Declarative (선언적)** | Scripted (스크립트) |
|---|---|---|
| 형태 | `pipeline { ... }` 로 시작하는 **정해진 틀** | 자유로운 Groovy 코드 |
| 난이도 | **쉬움** | 어려움 |
| 유연성 | 제한적 | 무엇이든 가능 |
| 권장 | **표준** ★ | 특수한 경우만 |

## Jenkinsfile 구조

```groovy
pipeline {
  /* 어디서 실행할 것인가 (any: 사용 가능한 아무 노드) */
  agent any

  /* 파이프라인 전역 변수 */
  environment {
    JDK_VERSION = '21'
  }

  /* 단계들 */
  stages {
    stage('Checkout') {
      steps {
        echo 'Checking out source code from Git...'
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'chmod +x gradlew'
        sh './gradlew clean bootJar -x test'
      }
    }

    stage('Test') {
      steps {
        sh './gradlew test'
      }
    }

    stage('Deploy') {
      steps {
        echo 'Deploying application to Target Server...'
      }
    }
  }

  /* 끝난 뒤 처리 */
  post {
    success { echo 'Pipeline Execution Succeeded!' }
    failure { echo 'Pipeline Execution Failed. Please check logs.' }
  }
}
```

### 블록별로 읽는 법

| 블록 | 뜻 |
|---|---|
| **`agent`** | **어디서 실행할까.** `any` = 아무 데나, `docker {...}` = 특정 컨테이너 안에서 |
| **`environment`** | 전역 **환경 변수** 정의 |
| **`stages` / `stage`** | 파이프라인의 **단계들.** UI에 단계별 진행 상황이 표시됨 |
| **`steps`** | 그 단계에서 실행할 **실제 명령들** |
| **`post`** | **끝난 뒤** 실행. `success`·`failure`·`always` 등 조건별 분기 |

### 자주 쓰는 step

| step | 하는 일 |
|---|---|
| `checkout scm` | **이 Jenkinsfile이 있는 저장소**를 체크아웃 (`scm` = Source Control Management) |
| `sh '...'` | **셸 명령 실행** (리눅스/맥). 윈도우는 `bat` |
| `echo '...'` | 로그 출력 |
| `withCredentials([...])` | **비밀값을 환경 변수로 잠깐 주입** (아래 참고) |
| `cleanWs()` | 작업 공간 정리 |

### `post`가 왜 유용한가

```groovy
post {
  always  { cleanWs() }              // 성공하든 실패하든 정리
  success { slackSend('배포 성공') }   // 성공했을 때만
  failure { slackSend('빌드 실패!') }  // 실패했을 때만
}
```

**`stages` 안에서 실패하면 그 뒤 단계는 건너뛰지만, `post`는 실행됩니다.**
그래서 **알림 전송과 뒷정리**를 여기에 둡니다.

## Credentials — 비밀값 관리

Jenkins는 비밀번호·토큰을 **자체 저장소에 암호화해 보관**하고, 파이프라인에서 **ID로 꺼내 씁니다.**

```groovy
withCredentials([usernamePassword(
    credentialsId: 'DOCKERHUB_CREDENTIALS',   // Jenkins 에 등록해 둔 ID
    usernameVariable: 'DOCKER_USER',           // 이 이름의 변수로 주입
    passwordVariable: 'DOCKER_PASS')]) {
  sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
}
```

**세 가지가 보장됩니다.**

| | |
|---|---|
| **코드에 비밀이 없다** | Jenkinsfile은 Git에 올라가므로 필수 |
| **블록 안에서만 유효** | `withCredentials` 밖으로 나가면 변수가 사라짐 |
| **로그에서 가려진다** | 실수로 출력해도 `****` 로 마스킹 |

> **`--password-stdin`을 쓰는 이유**: `docker login -p 비밀번호` 라고 쓰면
> **프로세스 목록(`ps`)에 비밀번호가 그대로 보입니다.** 표준 입력으로 넘기면 안 보입니다.

## 실행 트리거

파이프라인을 언제 돌릴지 정하는 방법들입니다.

| 방식 | 동작 | 특징 |
|---|---|---|
| **수동 실행** | 버튼 클릭 | 최초 검증용 |
| **폴링(Poll SCM)** | Jenkins가 **주기적으로 Git을 확인** | 설정 쉬움. **지연이 있고 Git에 부하** |
| **Webhook** ★ | Git이 **변경을 Jenkins에 알려줌** | **즉시 반응, 부하 없음** |
| 정기 실행(cron) | 정해진 시각에 | 야간 전체 빌드 등 |

### 기초 — Webhook이란

> **어떤 일이 생겼을 때, 미리 등록해 둔 주소로 알림을 쏴 주는 것**

```
   [ 폴링 ]   Jenkins ──"바뀐 거 있어?"──▶ GitHub   (1분마다 계속)
                      ◀──"아니"──
                      → 대부분 헛수고

   [ Webhook ] GitHub ──"방금 push 됐어!"──▶ Jenkins   (생겼을 때만)
```

**[observation/03의 Pull vs Push](../observation/03-prometheus와-grafana.md)와 같은 구도**입니다.

> ⚠️ **Webhook에는 전제가 있습니다.**
> **GitHub이 Jenkins에 접속할 수 있어야** 합니다.
> 내 노트북에서 도는 Jenkins는 **인터넷에서 보이지 않으므로** 그대로는 안 됩니다.
> → [실습 2](06-실습-jenkins와-argocd.md)에서 이 문제를 다룹니다.

## Jenkins에서 도커를 쓰는 방법

CI가 **도커 이미지를 빌드**하려면 Jenkins 안에서 도커 명령을 쓸 수 있어야 합니다.
**Jenkins 자체가 컨테이너면** 문제가 생깁니다 — 컨테이너 안에는 도커가 없으니까요.

**가장 흔한 해법: 호스트의 도커를 빌려 쓰기**

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

> **`/var/run/docker.sock`** 은 **도커 엔진과 대화하는 통로(소켓 파일)** 입니다.
> 이걸 컨테이너 안에 넣어 주면, **컨테이너 안의 도커 CLI가 호스트의 도커 엔진에 명령**할 수 있습니다.
>
> 이 방식을 **DooD (Docker outside of Docker)** 라고 부릅니다.
> 컨테이너 **안에** 도커를 설치하는 게 아니라, **밖의 도커를 쓰는 것**이라는 뜻입니다.

```
   [ Jenkins 컨테이너 ]
        docker build ...
             │  소켓을 통해 명령 전달
             ▼
   [ 호스트의 도커 엔진 ]  ← 여기서 실제로 이미지가 만들어진다
```

> ⚠️ **보안상 강력한 권한입니다.** 도커 소켓에 접근할 수 있으면
> **호스트의 어떤 컨테이너든 만들고 지울 수 있습니다.** 사실상 호스트 루트 권한과 같습니다.
> **실습에서는 편의상 쓰지만, 운영에서는 신중해야 합니다.**

---

[← 실습 1](04-실습-github-actions와-argocd.md) | [실습 2 — Jenkins와 ArgoCD →](06-실습-jenkins와-argocd.md)
