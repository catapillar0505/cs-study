> [← serverless 목차](README.md) · [aws](../README.md)

# API Gateway와 프록시 통합

## 한 줄로

Lambda는 HTTP를 모른다. **API Gateway가 HTTP 요청을 JSON 이벤트로 바꿔서 Lambda에 넘기고, Lambda가 돌려준 JSON을 다시 HTTP 응답으로 바꾼다.** 이 둘 사이의 약속이 "프록시 통합"이다.

## 왜 필요한가

02편의 함수들은 **콘솔 "테스트" 버튼**이나 SDK로만 부를 수 있었습니다. 브라우저나 `curl`로 부르려면 **HTTP 주소**가 있어야 합니다.

```
curl https://abc123.execute-api.ap-northeast-2.amazonaws.com/Prod/welcome?name=김철수
   │
   ▼
[ API Gateway ]   ← HTTPS 종료, 경로·메서드 매칭, (선택) 인증·쓰로틀링
   │ 이벤트 JSON
   ▼
[ Lambda ]
```

> Spring Cloud Gateway 같은 **애플리케이션 게이트웨이**와 하는 일이 겹칩니다(라우팅, 인증, 요청 제한).
> 차이는 **서버를 내가 안 띄운다**는 것. → [msa/03 API 게이트웨이](../../../../msa/03-api-게이트웨이.md)

## 프록시 통합 — 요청이 이렇게 바뀐다

**요청 전체를 가공 없이 통째로** Lambda에 넘기는 방식입니다. (`/{proxy+}` 같은 경로도 이 방식)

```
GET /welcome?name=김철수
User-Agent: curl/8.5.0
```

⬇ API Gateway가 만든 이벤트 (일부)

```json
{
  "httpMethod": "GET",
  "path": "/welcome",
  "queryStringParameters": { "name": "김철수" },
  "headers": { "User-Agent": "curl/8.5.0", "Host": "..." },
  "pathParameters": null,
  "body": null,
  "isBase64Encoded": false,
  "requestContext": { "stage": "Prod", "requestId": "...", "identity": { "sourceIp": "..." } }
}
```

이 JSON을 Java 객체로 받는 타입이 `aws-lambda-java-events` 라이브러리에 있습니다.

```groovy
implementation 'com.amazonaws:aws-lambda-java-events:3.16.1'
```

## 핸들러 — `api-gateway-basic`

```java
public class WelcomeHandler
    implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

  public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
    String method = input.getHttpMethod();
    Map<String, String> queryParams = input.getQueryStringParameters();
    Map<String, String> headers = input.getHeaders();
    ...
    APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent();
    response.setStatusCode(200);
    response.setBody("이름:" + name + ", 메서드: " + method + ", User-Agent: " + userAgent);
    return response;
  }
}
```

| 이벤트에서 꺼내기 | 메서드 |
|---|---|
| HTTP 메서드 | `getHttpMethod()` |
| 쿼리 스트링 `?a=1` | `getQueryStringParameters()` → `Map` 또는 **null** |
| 경로 변수 `/users/{id}` | `getPathParameters()` → `Map` 또는 **null** |
| 헤더 | `getHeaders()` |
| 본문 | `getBody()` → **문자열**. JSON이면 직접 파싱 |

응답 객체는 setter 방식과 **빌더처럼 쓰는 `with…` 방식** 둘 다 됩니다.

```java
return new APIGatewayProxyResponseEvent()
    .withStatusCode(200)
    .withHeaders(Map.of("Content-Type", "application/json; charset=UTF-8"))
    .withBody("{\"message\": \"ok\"}");
```

## 응답 모양이 틀리면 502

프록시 통합에서 Lambda는 **반드시 이 모양**을 돌려줘야 합니다.

```json
{ "statusCode": 200, "headers": { ... }, "body": "문자열" }
```

| Lambda가 한 일 | 클라이언트가 받는 것 |
|---|---|
| 올바른 모양 반환 | 그 `statusCode` 그대로 |
| **`null` 반환** / 모양이 다름 | **502 Bad Gateway** (Malformed Lambda proxy response) |
| 예외를 던짐 | **502** |
| Timeout 초과 | **504** (API Gateway 쪽 제한은 기본 29초) |

> **502는 "Lambda 코드가 에러를 냈다"가 아니라 "약속한 JSON 모양이 아니다"** 일 때도 납니다.
> `body`는 **객체가 아니라 문자열**이어야 한다는 걸 자주 놓칩니다.

## 흔한 실수

### ① `null` 체크를 절반만 함

```java
String name = "홍길동";
if (queryParams != null) {
  name = queryParams.get("name");   // ?age=20 처럼 name 없이 오면 → null
}
```

쿼리가 **아예 없으면** `queryParams` 자체가 `null`이고, **있는데 `name`만 없으면** `get()`이 `null`입니다. 두 경우를 다 막아야 합니다.

```java
String name = Optional.ofNullable(input.getQueryStringParameters())
    .map(q -> q.get("name"))
    .orElse("홍길동");
```

### ② 헤더 대소문자

HTTP 헤더 이름은 원래 **대소문자를 구분하지 않지만**, `Map.get("User-Agent")`는 구분합니다.
REST API(v1)는 보낸 대로 넘겨주지만, **HTTP API(v2) 이벤트는 헤더 이름을 전부 소문자로** 바꿉니다. 여러 형태로 올 수 있다고 생각하고 읽어야 합니다.

### ③ 한글이 깨짐

본문에 한글을 넣으면 `Content-Type`에 **`charset=UTF-8`** 을 명시해야 브라우저가 제대로 보여줍니다.

## REST API와 HTTP API

API Gateway에는 종류가 둘 있고, SAM의 `Type: Api`는 **REST API**를 만듭니다.

| | REST API (v1) | HTTP API (v2) |
|---|---|---|
| SAM 이벤트 타입 | `Api` | `HttpApi` |
| Java 이벤트 클래스 | `APIGatewayProxyRequestEvent` | `APIGatewayV2HTTPEvent` |
| 기능 | 많음 (API 키, 사용량 계획, 요청 검증 ...) | 적음 |
| 가격·지연 | 더 비쌈 | **더 싸고 빠름** |

**이벤트 JSON 모양이 다르므로** 핸들러의 입력 타입도 맞춰야 합니다.

## 스테이지 — URL의 `/Prod`

```
https://{api-id}.execute-api.{region}.amazonaws.com/Prod/welcome
                                                    └스테이지┘
```

같은 API를 `dev`, `prod` 처럼 **여러 버전으로 배포해 두는 단위**입니다. SAM이 자동으로 만드는 API는 스테이지 이름이 `Prod`입니다. → [04](04-sam과-cloudformation.md)

---

[← Java 핸들러와 Fat JAR](02-java-핸들러와-fat-jar.md) | [SAM과 CloudFormation →](04-sam과-cloudformation.md)
