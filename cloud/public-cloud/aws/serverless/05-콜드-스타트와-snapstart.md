> [← serverless 목차](README.md) · [aws](../README.md)

# 콜드 스타트와 SnapStart

## 한 줄로

실행 환경이 새로 뜰 때 드는 준비 시간이 **콜드 스타트**다. Java는 JVM 기동·클래스 로딩 때문에 특히 길다. **SnapStart는 준비가 끝난 상태를 스냅샷으로 찍어 두고, 다음부터는 그걸 복원해서** 준비 과정을 건너뛴다.

## 콜드 스타트의 구성

```
[ 콜드 스타트 ]
 ├─ ① 실행 환경(microVM) 생성, 코드 다운로드     ← AWS 몫
 ├─ ② Init 단계
 │     - JVM 기동
 │     - 핸들러 클래스 로딩
 │     - static 필드 초기화  (SDK 클라이언트 생성 등)   ← 내 코드
 └─ ③ Invoke 단계: handleRequest() 실행

[ 웜 스타트 ]
 └─ ③ 만 실행
```

Java에서 느린 곳은 **②** 입니다. 스프링처럼 클래스가 많으면 수 초가 걸리기도 합니다.

## 대책 ① — 비싼 건 핸들러 밖에서 한 번만

`sam-s3`의 핸들러:

```java
public class ProfileHandler implements RequestHandler<...> {

  // Init 단계에서 한 번만 만들어지고, 웜 스타트에서는 재사용됨
  private static final S3Client s3Client = S3Client.builder()
      .httpClientBuilder(UrlConnectionHttpClient.builder())
      .region(Region.AP_NORTHEAST_2)
      .build();
  private static final S3Presigner s3Presigner = S3Presigner.builder()
      .s3Client(s3Client)
      .build();

  public APIGatewayProxyResponseEvent handleRequest(...) {
    // 여기선 만들어진 클라이언트를 쓰기만
  }
}
```

| 위치 | 언제 실행 | 무엇을 두나 |
|---|---|---|
| **static / 생성자** | 환경당 **한 번** (Init) | SDK 클라이언트, 커넥션, 설정 읽기 |
| **handleRequest 안** | **요청마다** | 요청별 로직 |

핸들러 안에서 `S3Client.builder()...build()`를 하면 **요청마다** 클라이언트를 새로 만듭니다.

### 가벼운 HTTP 클라이언트 고르기

```groovy
implementation 'software.amazon.awssdk:s3'
implementation 'software.amazon.awssdk:url-connection-client'
```

AWS SDK v2는 HTTP 클라이언트를 고를 수 있습니다. **`UrlConnectionHttpClient`는 JDK 내장 `HttpURLConnection`을 써서 가볍고 초기화가 빠릅니다.** 기본값인 Apache 클라이언트보다 기능은 적지만, 요청을 몇 개 안 보내는 Lambda에는 이쪽이 유리합니다.

> 더 줄이려면 `s3` 의존성에서 기본 HTTP 클라이언트(apache·netty)를 `exclude`해서 **JAR 크기와 클래스 로딩**을 줄일 수 있습니다.

## 대책 ② — 메모리를 올리면 CPU도 오른다

Lambda는 **CPU를 따로 고를 수 없고, 메모리에 비례해서** 줍니다. (약 1,769MB에서 vCPU 1개 분량)

```yaml
ProfileFunction:
  Properties:
    MemorySize: 2048    # Globals 의 512 를 덮어씀
```

| | 512MB | 2048MB |
|---|---|---|
| CPU | 약 0.3 vCPU | 1 vCPU 이상 |
| JVM 기동·JIT | 느림 | **빠름** |
| GB-초 단가 | 싸다 | 4배 |
| 총 요금 | 오래 돌아서 **생각보다 안 쌀 수 있음** | 빨리 끝나서 **생각보다 안 비쌀 수 있음** |

Java Lambda에서 메모리를 크게 잡는 건 **메모리가 필요해서가 아니라 CPU가 필요해서**인 경우가 많습니다.

## 대책 ③ — SnapStart

### 발상

```
[ 배포할 때 (버전 발행 시) ]
  Init 단계 실행 → 메모리·디스크 상태를 통째로 스냅샷 → 저장

[ 콜드 스타트 때 ]
  Init 대신 스냅샷을 복원 → 바로 Invoke
```

**"JVM 띄우고 클래스 로딩하고 클라이언트 만드는 것"까지 끝난 상태에서 시작**합니다.

### 템플릿

```yaml
ProfileFunction:
  Type: AWS::Serverless::Function
  Properties:
    AutoPublishAlias: live            # ② 배포할 때마다 새 버전 발행 + 'live' 별칭이 가리키게
    SnapStart:
      ApplyOn: PublishedVersions      # ① 발행된 버전에만 스냅샷 적용
```

**둘이 꼭 같이 있어야 합니다.**

| 개념 | 뜻 |
|---|---|
| `$LATEST` | 편집 가능한 최신 코드. **SnapStart 적용 안 됨** |
| 버전 (1, 2, 3 …) | 발행 시점에 **고정된 스냅샷**. 바뀌지 않음 |
| 별칭 (`live`) | 특정 버전을 가리키는 **이름표**. 배포 때 새 버전으로 옮겨짐 |

SnapStart는 **"고정된 코드"에만 스냅샷을 찍을 수 있으니** 버전이 필요하고, `AutoPublishAlias`가 매 배포마다 버전을 발행해줍니다. SAM이 만드는 API Gateway는 **별칭(`live`)을 호출**하도록 연결됩니다.

> 버전 발행 시 스냅샷을 만드느라 **배포가 몇 분 더 걸립니다.**

### 스냅샷의 함정 — 복원된 건 "과거의 나"

한 스냅샷에서 **여러 환경이 복원**됩니다. Init에서 만든 것이 **모두에게 똑같이 복사**된다는 뜻입니다.

| Init에서 만든 것 | 문제 |
|---|---|
| 난수 시드, UUID | **모든 환경이 같은 값**을 가질 수 있음 |
| 네트워크 연결 (DB 커넥션) | 스냅샷 후 시간이 지나 **끊겨 있음** |
| 현재 시각, 임시 자격 증명 | **스냅샷 시점 값**에 멈춰 있음 |

→ 이런 건 **핸들러 안에서** 만들거나, 복원 후 실행되는 **런타임 훅**(CRaC `afterRestore`)에서 다시 준비합니다.
`S3Client`처럼 요청 때마다 연결을 맺는 클라이언트는 스냅샷에 넣어도 괜찮습니다.

### 다른 방법과 비교

| 방법 | 원리 | 비용 |
|---|---|---|
| static 초기화 | 웜 스타트에서 재사용 | 없음 |
| 메모리 증가 | CPU가 늘어 Init 자체가 빨라짐 | 단가 증가 |
| **SnapStart** | Init을 건너뜀 | Java는 추가 요금 없음 |
| Provisioned Concurrency | 환경을 **미리 띄워 둠** → 콜드 스타트 자체가 없음 | **켜둔 시간만큼 과금** |

> SnapStart와 Provisioned Concurrency는 **같이 쓸 수 없습니다.**

---

[← SAM과 CloudFormation](04-sam과-cloudformation.md) | [실습 — S3 Presigned URL →](06-실습-s3-presigned-url.md)
