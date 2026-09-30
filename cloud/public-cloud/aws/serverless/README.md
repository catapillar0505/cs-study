# serverless — AWS 서버리스

EC2처럼 **서버를 빌려서 그 위에 앱을 올리는** 방식이 아니라,
**함수 하나만 올려두고 요청이 올 때만 실행되게** 하는 방식을 정리합니다.

Java 21로 Lambda 함수를 콘솔에 직접 올려보는 것에서 시작해서,
API Gateway로 HTTP 요청을 받고, 마지막에는 **SAM으로 인프라까지 코드로 배포**하는 순서로 쌓았습니다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [서버리스와 Lambda](01-서버리스와-lambda.md) | 서버가 없다는 말의 뜻, 실행 모델, 과금, 제약, EC2·컨테이너와의 비교 |
| 02 | [Java 핸들러와 Fat JAR](02-java-핸들러와-fat-jar.md) | `RequestHandler<I, O>`, 입력 역직렬화, 환경 변수, **Shadow로 의존성 묶기**, 계산기 실습 |
| 03 | [API Gateway와 프록시 통합](03-api-gateway와-프록시-통합.md) | HTTP → 이벤트 변환, `APIGatewayProxyRequestEvent`, 응답 형식, 502가 나는 이유 |
| 04 | [SAM과 CloudFormation](04-sam과-cloudformation.md) | `template.yaml` 읽는 법, 암묵적 리소스, `samconfig.toml`, build → deploy → delete |
| 05 | [콜드 스타트와 SnapStart](05-콜드-스타트와-snapstart.md) | Init 단계, 메모리 = CPU, static 초기화, **스냅샷으로 Init 건너뛰기** |
| 06 | [실습 — S3 Presigned URL](06-실습-s3-presigned-url.md) | 파일을 Lambda로 받지 않는 이유, 서명은 로컬 계산, CORS, IAM 정책 템플릿 |

## 전체 그림

```
 클라이언트 (curl / 브라우저)
   │ HTTPS
   ▼
[ API Gateway ]  ── HTTP 요청을 JSON 이벤트로 바꿔서
   │                  (프록시 통합)
   ▼
[ Lambda ]  ── Java 핸들러 handleRequest(event, context)
   │           요청이 올 때만 실행 환경이 뜬다
   │
   ├─ 환경 변수 (BUCKET_NAME ...)
   ├─ 실행 역할 (IAM Role) ── 무엇을 호출할 수 있는지
   └─▶ [ S3 ] 등 다른 AWS 서비스

 ─────────────────────────────────────────────
 위 전부를 template.yaml 한 파일로 선언  ── SAM
   sam build → sam deploy → CloudFormation 스택
```

## 실습 프로젝트 순서

| 프로젝트 | 배포 방식 | 배운 것 | 문서 |
|---|---|---|---|
| `lambda-basic` | 콘솔에 JAR 업로드 | 핸들러 기본형, `Map` 입력, 로거 | [02](02-java-핸들러와-fat-jar.md) |
| `lambda-calculator` | 콘솔에 JAR 업로드 | `record` 입력, 환경 변수, 예외 처리 | [02](02-java-핸들러와-fat-jar.md) |
| `api-gateway-basic` | 콘솔 + API Gateway 연결 | 프록시 이벤트, 쿼리·헤더 읽기, 응답 객체 | [03](03-api-gateway와-프록시-통합.md) |
| `sam-basic` | **SAM** | 템플릿, 암묵적 API·Role, 로컬 실행 | [04](04-sam과-cloudformation.md) |
| `sam-s3` | **SAM** | S3 버킷·정책까지 코드로, SnapStart, Presigned URL | [05](05-콜드-스타트와-snapstart.md) · [06](06-실습-s3-presigned-url.md) |

## 이 폴더를 관통하는 문장들

> **서버리스는 서버가 없는 게 아니라, 서버를 내가 관리하지 않는 것이다.**

> **Lambda는 요청마다 새로 뜨는 게 아니다.** 한 번 뜬 실행 환경은 재사용된다.
> 그래서 **비싼 초기화는 핸들러 밖(static)에 둔다.**

> **API Gateway와 Lambda 사이의 약속은 JSON 모양이다.** 모양이 틀리면 코드가 멀쩡해도 502가 난다.

> **콘솔 클릭은 기록이 남지 않는다.** 템플릿으로 선언해야 똑같이 다시 만들고, 한 번에 지울 수 있다.

> **큰 데이터는 Lambda를 거치지 않게 한다.** Lambda는 "허가증"만 만들어 주고, 파일은 S3로 직접 간다.

## 읽는 순서

```
01 서버리스와 Lambda        "무엇이 다른가"
   ↓
02 Java 핸들러              "함수 하나를 올려본다"  (콘솔)
   ↓
03 API Gateway              "HTTP로 부를 수 있게"
   ↓
04 SAM                      "클릭을 코드로"
   ↓
05 콜드 스타트·SnapStart     "Java가 느리게 뜨는 문제"
   ↓
06 실습 — Presigned URL     전부 합친 구성
```

## 함께 보기

- Lambda를 VPC 안에 넣으면 인터넷에 나가려면 NAT가 필요하다 → [aws/05 NAT Gateway](../05-nat-gateway.md)
- 오브젝트 스토리지(S3)가 뭔지 → [storage/05](../../../../storage/05-aws-스토리지-ebs-efs-s3.md)
- 애플리케이션 층의 API 게이트웨이(Spring Cloud Gateway) → [msa/03](../../../../msa/03-api-게이트웨이.md) · [msa/gateway](../../../../msa/gateway/README.md)
- 브라우저가 다른 출처로 요청할 때(CORS) → [msa/auth/06](../../../../msa/auth/06-spa와-cors.md)
- 인프라를 코드로 관리하는 다른 도구 → [devops/terraform](../../../../devops/terraform/README.md)
- 이벤트가 오면 실행된다는 발상 → [msa/eda](../../../../msa/eda/README.md)
- 컨테이너와 격리 기술 → [container/01](../../../../container/01-namespace와-cgroup.md)
