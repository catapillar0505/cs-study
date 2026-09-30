> [← serverless 목차](README.md) · [aws](../README.md)

# 실습 — S3 Presigned URL

## 한 줄로

프로필 이미지를 올릴 때 **파일을 Lambda로 받지 않는다.** Lambda는 "이 경로에 5분 동안 PUT해도 된다"는 **서명된 URL만 만들어 주고**, 클라이언트가 그 URL로 **S3에 직접** 올린다.

## 왜 Lambda로 파일을 안 받나

```
❌ 클라이언트 ──(이미지 5MB)──▶ API Gateway ──▶ Lambda ──▶ S3
```

| 문제 | 이유 |
|---|---|
| **크기 제한** | API Gateway 요청 본문 10MB, Lambda 동기 호출 페이로드 6MB |
| **돈** | 파일이 지나가는 동안 Lambda가 **기다리는 시간도 과금** |
| **인코딩** | 바이너리를 Base64로 바꿔서 넘겨야 함 (크기 약 1.33배) |

```
✅ ① 클라이언트 ──"업로드할게요"──▶ API Gateway ──▶ Lambda
   ② Lambda ──"이 URL로 5분 안에 올려"──▶ 클라이언트
   ③ 클라이언트 ──(이미지)──▶ S3   ← Lambda를 안 거침
```

**Lambda는 허가증만 발급하고, 무거운 짐은 S3가 직접 받습니다.**

## 프로젝트 구조

```
sam-s3
├── template.yaml          # 버킷 + 함수 + API + 권한
├── samconfig.toml
├── events/event.json
├── QR.jpg                 # 업로드 테스트용 이미지
└── s3Function/
    ├── build.gradle
    └── src/main/java/com/example/user/ProfileHandler.java
```

---

## Step 1. 템플릿 — 버킷까지 코드로

```yaml
Resources:
  ProfileBucket:
    Type: AWS::S3::Bucket                 # SAM 축약형이 아닌 CloudFormation 리소스도 그대로 씀
    Properties:
      BucketName: !Sub "simple-profile-bucket-${AWS::AccountId}-teacher-min"
      CorsConfiguration:
        CorsRules:
          - AllowedHeaders: ['*']
            AllowedOrigins: ['*']
            AllowedMethods: [GET, POST, PUT, DELETE, HEAD]

  ProfileFunction:
    Type: AWS::Serverless::Function
    Properties:
      Policies:
        - S3CrudPolicy:                   # SAM 정책 템플릿
            BucketName: !Ref ProfileBucket
      AutoPublishAlias: live
      SnapStart:
        ApplyOn: PublishedVersions
      CodeUri: s3Function
      Handler: com.example.user.ProfileHandler::handleRequest
      Runtime: java21
      MemorySize: 2048
      Environment:
        Variables:
          BUCKET_NAME: !Ref ProfileBucket # 버킷 이름을 코드에 하드코딩하지 않음
      Events:
        ApiGatewayRestApi:
          Type: Api
          Properties:
            Path: /api/users/profile/getPresignedUrl
            Method: post
```

### 하나씩 보면

**버킷 이름에 `${AWS::AccountId}`를 넣은 이유**
S3 버킷 이름은 **전 세계에서 유일**해야 합니다. 같은 템플릿을 여러 사람이 배포해도 안 겹치게 계정 ID를 붙입니다.

**`!Ref ProfileBucket` → 환경 변수**
템플릿 안에서 만든 버킷의 **실제 이름을 배포 시점에 함수에 꽂아줍니다.** 코드는 `System.getenv("BUCKET_NAME")`만 읽으면 되고, 버킷 이름이 바뀌어도 코드는 그대로입니다.

**`S3CrudPolicy` — 정책 템플릿**
IAM 정책 JSON을 직접 쓰지 않고 **"이 버킷에 대해 읽기·쓰기·삭제"** 권한을 한 줄로 붙입니다. SAM이 암묵적 Role(`ProfileFunctionRole`)에 정책을 달아줍니다.

| 정책 템플릿 | 권한 |
|---|---|
| `S3ReadPolicy` | 읽기만 |
| `S3WritePolicy` | 쓰기만 |
| `S3CrudPolicy` | 읽기·쓰기·삭제 |

> **최소 권한 원칙**으로 보면 이 실습은 PUT만 하니 `S3WritePolicy`로 충분합니다.

**CORS 설정**
**브라우저**가 `https://내사이트.com`에서 `https://버킷.s3...amazonaws.com`으로 PUT하면 **출처가 다르므로** 브라우저가 먼저 S3에 "허용하냐"고 묻습니다(preflight). 버킷에 CORS 규칙이 없으면 **브라우저가 막습니다.**
`curl`은 CORS를 검사하지 않으니 테스트할 때는 없어도 됩니다. → [msa/auth/06 SPA와 CORS](../../../../msa/auth/06-spa와-cors.md)

> `AllowedOrigins: ['*']`는 실습용입니다. 운영에서는 **내 프론트엔드 도메인만** 적습니다.

## Step 2. 핸들러

```java
public class ProfileHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

  private static final S3Client s3Client = S3Client.builder()
      .httpClientBuilder(UrlConnectionHttpClient.builder())
      .region(Region.AP_NORTHEAST_2)
      .build();
  private static final S3Presigner s3Presigner = S3Presigner.builder()
      .s3Client(s3Client)
      .build();

  private final String BUCKET_NAME = System.getenv("BUCKET_NAME");
  private final String KEY = "QR.jpg";

  public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
    // ① 어떤 요청을 허가할지 — 버킷, 키, Content-Type
    PutObjectRequest objectRequest = PutObjectRequest.builder()
        .bucket(BUCKET_NAME)
        .key(KEY)
        .contentType("image/jpeg")
        .build();

    // ② 얼마 동안 유효한지
    PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(5))
        .putObjectRequest(objectRequest)
        .build();

    // ③ 서명된 URL 생성
    String uploadUrl = s3Presigner.presignPutObject(presignRequest).url().toString();

    return new APIGatewayProxyResponseEvent()
        .withStatusCode(200)
        .withHeaders(Map.of("Content-Type", "application/json; charset=UTF-8"))
        .withBody("{\"uploadUrl\": \"" + uploadUrl + "\"}");
  }
}
```

`static` 클라이언트와 `UrlConnectionHttpClient`를 쓴 이유는 → [05 콜드 스타트](05-콜드-스타트와-snapstart.md)

## Presigned URL은 어떻게 동작하나

### 받은 URL의 모양

```
https://simple-profile-bucket-123456789012-teacher-min.s3.ap-northeast-2.amazonaws.com/QR.jpg
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=ASIA.../20260929/ap-northeast-2/s3/aws4_request   ← 누구의 자격 증명으로
  &X-Amz-Date=20260929T051000Z                                         ← 언제 서명했고
  &X-Amz-Expires=300                                                   ← 몇 초 유효한지
  &X-Amz-SignedHeaders=content-type;host                               ← 어떤 헤더까지 서명에 포함했는지
  &X-Amz-Security-Token=...
  &X-Amz-Signature=3f2a...                                             ← 위 전부를 비밀키로 서명한 값
```

### 서명은 로컬 계산이다

```
Lambda                                   S3
  │ presignPutObject()                    │
  │  = 메서드·경로·헤더·만료시각을            │
  │    Lambda 역할의 비밀키로 HMAC 계산      │   ← 네트워크 요청 없음!
  │                                        │
  └── URL 반환                              │
                                           │
클라이언트 ── PUT (URL + 파일) ─────────────▶│  S3가 같은 계산을 다시 해서
                                           │  서명이 맞는지, 만료 안 됐는지,
                                           │  서명한 주체에게 PutObject 권한이 있는지 확인
```

| 흔한 오해 | 실제 |
|---|---|
| "presign 할 때 S3에 요청이 간다" | **안 간다.** 순수 계산이다. 버킷이 없어도 URL은 만들어진다 |
| "URL에 권한이 들어 있다" | URL에는 **서명만** 있다. 권한은 **사용하는 순간** S3가 서명한 주체(Lambda Role)의 권한을 확인한다 |
| "`Duration.ofMinutes(5)`면 무조건 5분 유효" | Lambda Role의 **임시 자격 증명이 먼저 만료되면** URL도 그때 죽는다 |

그래서 **Lambda Role에 `S3CrudPolicy`가 없으면** URL은 잘 만들어지는데 **업로드할 때 403**이 납니다.

## Step 3. 배포와 테스트

```bash
cd sam-s3
sam build
sam deploy          # 첫 배포라면 sam deploy --guided
```

출력된 `ApiGatewayRestApi` 주소로 URL을 받습니다.

```bash
curl -X POST https://{api-id}.execute-api.ap-northeast-2.amazonaws.com/Prod/api/users/profile/getPresignedUrl/
# {"uploadUrl": "https://simple-profile-bucket-...?X-Amz-..."}
```

받은 URL로 **S3에 직접** 올립니다. **Content-Type을 서명할 때와 똑같이** 보내야 합니다.

```bash
curl -X PUT -T QR.jpg -H "Content-Type: image/jpeg" "받은_uploadUrl"
```

확인:

```bash
aws s3 ls s3://simple-profile-bucket-{계정ID}-teacher-min/
```

## 자주 만나는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| PUT이 **403 SignatureDoesNotMatch** | `Content-Type`을 안 보냈거나 다르게 보냄 (서명에 포함됨) | 서명할 때와 **같은 값**으로 |
| PUT이 **403 AccessDenied** | Lambda Role에 PutObject 권한 없음 | `Policies`에 S3 정책 추가 |
| PUT이 **403 Request has expired** | 5분 지남 / 임시 자격 증명 만료 | URL 다시 발급 |
| URL의 `&`가 잘려서 이상한 요청 | 셸에서 URL을 **따옴표 없이** 씀 | `"..."`로 감싸기 |
| 브라우저에서만 실패 (curl은 됨) | CORS 규칙 없음 / Origin 불일치 | 버킷 `CorsConfiguration` 확인 |
| API 호출이 **502** | 핸들러가 `null` 반환 / 환경 변수 누락으로 예외 | [03](03-api-gateway와-프록시-통합.md) 응답 형식 확인 |
| `sam delete` 실패 | **버킷이 비어 있지 않음** | `aws s3 rm s3://버킷 --recursive` 후 다시 |

## 이 구성의 한계 — 실무로 가려면

| 한계 | 설명 | 개선 |
|---|---|---|
| **키가 고정** (`QR.jpg`) | 누가 올려도 **같은 파일을 덮어씀** | `users/{userId}/{UUID}.jpg` 처럼 사용자·요청별 키 |
| **누구나 URL 발급 가능** | API에 인증이 없음 | API Gateway Authorizer / Cognito로 사용자 확인 |
| **파일 검증 없음** | 이미지가 아닌 걸 올려도, 100MB를 올려도 막을 수 없음 | Presigned **POST** + 조건(`content-length-range`)으로 크기 제한 |
| **올라갔는지 모름** | Lambda는 URL만 주고 끝. 업로드 성공 여부를 모름 | S3 이벤트 알림 → 다른 Lambda가 후처리(썸네일, DB 기록) |
| 메서드 체크 후 `null` 반환 | API가 POST만 연결돼 있어 실제로는 안 탐. 탔다면 502 | `405` 응답을 명시적으로 반환 |
| 리전 하드코딩 | `Region.AP_NORTHEAST_2` | 환경 변수 `AWS_REGION`(Lambda가 자동 제공) 사용 |

> 마지막 "S3 이벤트 → Lambda" 구성은 **업로드라는 이벤트가 다음 처리를 깨우는** 전형적인 이벤트 기반 구조입니다. → [msa/eda](../../../../msa/eda/README.md)

## 정리

```
template.yaml 한 파일로
  ├─ S3 버킷 (+ CORS)
  ├─ Lambda 함수 (+ SnapStart, 버전·별칭)
  ├─ 실행 역할 (+ S3 권한)          ← 암묵적 생성 + 정책 템플릿
  └─ API Gateway (POST 경로)        ← 암묵적 생성
을 만들고, sam delete 한 번으로 전부 지운다.
```

**Lambda는 "누가 무엇을 올려도 되는지" 판단만 하고, 데이터는 가장 가까운 곳(S3)으로 바로 보낸다.** 서버리스에서 큰 파일을 다루는 기본 패턴입니다.

---

[← 콜드 스타트와 SnapStart](05-콜드-스타트와-snapstart.md)
