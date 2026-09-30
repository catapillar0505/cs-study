> [← serverless 목차](README.md) · [aws](../README.md)

# SAM과 CloudFormation

## 한 줄로

**콘솔에서 클릭하던 것(함수 생성, API 연결, 권한 부여)을 `template.yaml` 한 파일에 선언하고, `sam deploy` 한 번으로 전부 만든다.** SAM은 CloudFormation을 서버리스용으로 짧게 쓰게 해주는 확장이다.

## 문제부터 — 콘솔로 만들면

03편까지 콘솔로 한 일을 세어보면

```
① Lambda 함수 생성, 런타임 java21 선택
② JAR 업로드, 핸들러 이름 입력
③ 메모리·타임아웃·환경 변수 설정
④ API Gateway 생성 → 리소스 → 메서드 → Lambda 연결 → 배포(스테이지)
⑤ API Gateway가 Lambda를 호출할 수 있게 권한 추가
⑥ Lambda 실행 역할(IAM Role) 생성
```

| 문제 | 어떻게 나타나나 |
|---|---|
| **기록이 없다** | 누가 언제 메모리를 바꿨는지 모름 |
| **똑같이 다시 못 만든다** | dev에서 된 걸 prod에 옮기려면 처음부터 다시 클릭 |
| **지우기 어렵다** | 함수는 지웠는데 API·Role·로그 그룹이 남아서 쌓임 |

→ **인프라를 코드로(IaC)** 선언하면 이 셋이 해결됩니다. Terraform과 같은 발상입니다. → [devops/terraform](../../../../devops/terraform/README.md)

## CloudFormation과 SAM의 관계

```
template.yaml (SAM 문법, 짧음)
     │  Transform: AWS::Serverless-2016-10-31
     ▼  ← AWS가 변환
CloudFormation 템플릿 (길고 자세함)
     │
     ▼
[ 스택 ]  = 이 템플릿으로 만든 리소스 묶음
   ├─ Lambda 함수
   ├─ IAM Role
   ├─ API Gateway (REST API, 스테이지, 권한)
   └─ ...
```

- **CloudFormation**: AWS의 IaC 엔진. 템플릿을 받아 리소스를 만들고, **스택 단위로 관리**합니다.
- **SAM**: `AWS::Serverless::Function` 같은 **축약형 리소스**를 제공합니다. 함수 하나 선언하면 Role·API·권한이 **자동으로 따라 생깁니다.**
- **SAM CLI**: 빌드·로컬 실행·배포를 해주는 명령어 도구 (`sam ...`)

## `template.yaml` 읽는 법 — `sam-basic`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31     # ← 이 줄이 있어야 SAM 문법이 해석됨

Globals:                                  # 모든 함수에 공통 적용
  Function:
    Timeout: 20
    MemorySize: 512

Resources:
  HelloWorldFunction:                     # 논리 ID (템플릿 안에서 부르는 이름)
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: myFunction                 # 소스 폴더 (build.gradle 있는 곳)
      Handler: helloworld.App::handleRequest
      Runtime: java21
      Architectures: [x86_64]
      Environment:
        Variables:
          PARAM1: VALUE
      Events:                             # 무엇이 이 함수를 깨우는가
        HelloWorld:
          Type: Api                       # → API Gateway REST API 자동 생성
          Properties:
            Path: /welcome
            Method: get

Outputs:                                  # 배포 후 화면에 출력할 값
  HelloWorldApi:
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.${AWS::URLSuffix}/Prod/welcome/"
  HelloWorldFunction:
    Value: !GetAtt HelloWorldFunction.Arn
  HelloWorldFunctionIamRole:
    Value: !GetAtt HelloWorldFunctionRole.Arn
```

### 선언하지 않았는데 생기는 것 — 암묵적 리소스

템플릿에는 함수 하나만 적었는데, `Outputs`에서 **`ServerlessRestApi`**, **`HelloWorldFunctionRole`** 을 참조하고 있습니다.

| 이름 | 무엇 | 왜 생기나 |
|---|---|---|
| `ServerlessRestApi` | REST API + `Prod` 스테이지 | `Events`에 `Type: Api`가 있어서 |
| `HelloWorldFunctionRole` | 함수 실행 역할 | 함수는 역할 없이 못 돔. 이름 = `{논리ID}Role` |
| (Lambda 권한) | API Gateway → Lambda 호출 허가 | 이벤트 소스가 호출할 수 있어야 해서 |

**이게 SAM을 쓰는 이유**입니다. CloudFormation으로 직접 쓰면 이것들을 전부 손으로 적어야 합니다.

### 자주 쓰는 내장 함수

| 함수 | 뜻 | 예 |
|---|---|---|
| `!Ref X` | X의 **기본값** (버킷이면 이름, 함수면 이름) | `!Ref ProfileBucket` |
| `!GetAtt X.속성` | X의 **특정 속성** | `!GetAtt HelloWorldFunction.Arn` |
| `!Sub "...${X}..."` | 문자열 안에 값 끼워넣기 | `!Sub "bucket-${AWS::AccountId}"` |

`AWS::Region`, `AWS::AccountId`, `AWS::URLSuffix`는 **배포 시점에 채워지는 가상 파라미터**입니다.

### Globals와 개별 설정

`Globals.Function`은 **모든 함수의 기본값**이고, 함수에 같은 속성을 적으면 **함수 쪽이 이깁니다.** `sam-s3`는 Globals에 512MB를 두고 함수에서 2048MB로 덮어썼습니다.

> `Timeout: 20` — API Gateway의 기본 응답 제한(29초)보다 **짧게** 둔 게 포인트입니다.
> Lambda가 29초 넘게 돌면 클라이언트는 이미 504를 받았는데 Lambda는 헛돌며 요금만 냅니다.

## `samconfig.toml` — 명령어 옵션을 파일로

`sam deploy --guided`로 처음 배포할 때 물어보는 값들을 저장해두는 파일입니다. 다음부터는 `sam deploy`만 치면 됩니다.

```toml
[default.deploy.parameters]
stack_name        = "sam-basic-teacher-min"   # CloudFormation 스택 이름
region            = "ap-northeast-2"
capabilities      = "CAPABILITY_IAM"          # IAM 리소스를 만들어도 된다는 동의
confirm_changeset = true                      # 뭐가 바뀌는지 보여주고 y/n 물어봄
resolve_s3        = true                      # 빌드 결과물 올릴 S3 버킷을 SAM이 알아서 만듦
s3_prefix         = "sam-basic-teacher-min"

[default.build.parameters]
cached   = true      # 안 바뀐 함수는 다시 빌드 안 함
parallel = true      # 함수 여러 개면 동시에 빌드
```

| 옵션 | 왜 필요한가 |
|---|---|
| `CAPABILITY_IAM` | 템플릿이 **IAM Role을 만들기 때문**. 권한을 만드는 건 위험하니 명시적 동의를 요구함. 없으면 배포 거부 |
| `resolve_s3` | CloudFormation은 코드를 **S3에 올린 뒤 그 주소로** 함수를 만든다. 그 버킷(`aws-sam-cli-managed-default`)을 자동 관리 |
| `confirm_changeset` | **변경 세트(ChangeSet)** = "추가/수정/삭제될 리소스 목록". 실수로 DB를 지우는 걸 막아줌 |

## 명령어 흐름

```bash
sam init                 # 템플릿으로 프로젝트 생성 (sam-basic 은 여기서 시작)
sam build                # .aws-sam/build/ 에 배포용 결과물 생성
sam local invoke HelloWorldFunction -e events/event.json   # 도커로 로컬 실행
sam local start-api      # 로컬에 API Gateway 흉내 (localhost:3000)
sam deploy --guided      # 첫 배포 (samconfig.toml 생성)
sam deploy               # 이후 배포
sam logs -n HelloWorldFunction --tail   # CloudWatch 로그 따라보기
sam delete               # 스택 통째로 삭제
```

### `sam build`가 하는 일 — Shadow가 필요 없는 이유

```
.aws-sam/build/HelloWorldFunction/
├── helloworld/App.class
└── lib/
    ├── aws-lambda-java-core-1.4.0.jar
    ├── aws-lambda-java-events-3.16.1.jar
    └── joda-time-2.10.8.jar
```

`sam build`가 Gradle로 컴파일하고 **의존성 JAR들을 `lib/`에 모아줍니다.** Lambda Java 런타임은 `lib/` 아래 JAR를 classpath에 올리므로 **Fat JAR를 따로 만들 필요가 없습니다.** 그래서 `sam-basic`의 `build.gradle`에는 Shadow 플러그인이 없습니다. → [02](02-java-핸들러와-fat-jar.md)

### `sam local` — 배포 전에 확인하기

`events/event.json`은 **API Gateway가 보낼 이벤트의 샘플**입니다. 이걸 넣고 로컬 도커 컨테이너에서 함수를 돌려봅니다.

- **Docker가 떠 있어야** 합니다 (Lambda 런타임 이미지를 받아서 실행).
- `samconfig.toml`의 `warm_containers = "EAGER"` → 로컬 컨테이너를 미리 띄워 두고 재사용해서 **매번 콜드 스타트하지 않게** 합니다.

## `sam-basic` 함수가 하는 일

```java
final String pageContents = this.getPageContents("https://checkip.amazonaws.com");
String output = String.format("{ \"message\": \"hello world\", \"location\": \"%s\" }", pageContents);
```

Lambda가 **밖으로 나갈 때 쓰는 공인 IP**를 확인해서 돌려줍니다. 호출할 때마다 IP가 바뀔 수 있는데, **Lambda는 AWS가 관리하는 공유 네트워크에서 돌기 때문**입니다.

> 이 함수를 **VPC 안에 넣으면** 기본으로는 인터넷에 못 나갑니다. 프라이빗 서브넷 + **NAT Gateway**가 필요합니다. → [aws/05](../05-nat-gateway.md)
> 고정 IP로 나가야 하는 요구(외부 API 화이트리스트)가 있으면 이 구성을 씁니다.

## 흔한 오해 · 문제

| 증상 / 오해 | 실제 |
|---|---|
| "`sam deploy`가 코드만 올린다" | **스택 전체**를 템플릿과 맞춘다. 템플릿에서 지운 리소스는 **실제로 삭제**된다 |
| `Requires capabilities: [CAPABILITY_IAM]` | `samconfig.toml`에 `capabilities` 누락 |
| `sam local`이 안 됨 | Docker 미실행 |
| 콘솔에서 고친 설정이 다음 배포 때 사라짐 | **템플릿이 진실**이다. 콘솔에서 고치면 드리프트가 생기고 다음 배포에 덮어써짐 |
| `sam delete`가 실패 | 스택이 만든 **S3 버킷에 파일이 남아 있으면** 삭제 불가. 비우고 다시 → [06](06-실습-s3-presigned-url.md) |
| `.aws-sam/`을 Git에 올림 | 빌드 결과물·캐시다. `.gitignore`에 넣는다 |

---

[← API Gateway와 프록시 통합](03-api-gateway와-프록시-통합.md) | [콜드 스타트와 SnapStart →](05-콜드-스타트와-snapstart.md)
