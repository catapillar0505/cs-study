> [← kubernetes 목차](README.md)

# Ingress — 클러스터의 L7 대문

## 문제: Service만으론 외부 노출이 비싸다

```
api.basecamp.com   → Service(LoadBalancer) → 클라우드 LB 1개 ($$$)
admin.basecamp.com → Service(LoadBalancer) → 클라우드 LB 1개 ($$$)
shop.basecamp.com  → Service(LoadBalancer) → 클라우드 LB 1개 ($$$)
```

서비스마다 LB 하나씩. 비용도, IP도, 인증서 관리 지점도 3배.

## 해결: 입구를 하나로 합치고 규칙으로 분배

```
                    ┌─ LoadBalancer 1개 ─┐
  모든 도메인  ────→ │  Ingress Controller │
                    └──────────┬──────────┘
                               │ 규칙 보고 분배
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      Service(api)     Service(admin)    Service(shop)
              │                │                │
            [Pod]            [Pod]            [Pod]
```

> **Spring 비유**: Ingress는 **`@RequestMapping` 라우팅 규칙표**를 애플리케이션이 아니라 **클러스터 입구에서** 적용하는 것. 또는 **Nginx 리버스 프록시 설정을 k8s 오브젝트화한 것** (실제 구현체가 대부분 nginx).

## Service vs Ingress — 계층이 다르다

| 오브젝트 | OSI 계층 | 아는 것 |
|---|---|---|
| **Service** | L4 (전송) | IP, 포트만 |
| **Ingress** | L7 (응용) | **도메인, URL 경로, HTTP 헤더** |

Service는 "10.96.0.42:80으로 온 패킷"까지만 안다. **URL을 모른다.** Ingress는 HTTP를 뜯어보므로 `/api/users`인지 `/admin`인지 구분한다.

## ★ Ingress는 두 조각이다 (핵심 함정)

| | 정체 | 역할 |
|---|---|---|
| **Ingress (리소스)** | YAML 규칙표 | "이 도메인은 저 Service로"라고 **적어둔 것** |
| **Ingress Controller** | 실제 도는 Pod (nginx 등) | 규칙을 읽고 **실제로 트래픽 처리** |

**Controller를 설치하지 않으면 Ingress YAML을 만들어도 아무 일도 일어나지 않는다.**

> **Java 비유**: Ingress = `interface`, Controller = 구현체. `List` 선언만으론 동작하지 않고 `ArrayList`를 넣어야 한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

## YAML 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basecamp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx        # 어느 Controller가 처리할지
  rules:
  - host: api.basecamp.com       # ① 도메인으로 분기
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service   # ← 워크로드 문서에서 만든 Service
            port:
              number: 80
      - path: /orders            # ② 경로로 분기
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
  tls:                            # ③ HTTPS를 여기서 종료
  - hosts:
    - api.basecamp.com
    secretName: basecamp-tls      # ← 인증서는 Secret에 (→ [ConfigMap / Secret](08-configmap과-secret.md))
```

## ★ 전체 트래픽 흐름 (문서 전체가 여기서 만난다)

```
브라우저: https://api.basecamp.com/users
   ↓ DNS
클라우드 LoadBalancer (외부 IP)
   ↓
Ingress Controller Pod (nginx)
   ↓ TLS 종료 + Host/Path 규칙 매칭          ← Ingress
Service: user-service (ClusterIP 10.96.0.42)
   ↓ CoreDNS 조회 + iptables DNAT            ← 서비스 디스커버리 (br_netfilter 필요)
Pod (10.244.2.11:8080)                       ← 워크로드, 라벨로 연결
   ↓ 환경변수로 주입된 DB 접속 정보          ← 오브젝트 구조
Spring Boot 앱
```

## Ingress가 대신 해주는 일

| 기능 | 내용 |
|---|---|
| **TLS 종료** | HTTPS를 여기서 풀고 내부는 HTTP (인증서 한 곳만 관리) |
| **가상 호스팅** | 도메인별 분기 |
| **경로 라우팅** | `/api` → A, `/admin` → B |
| **rate limiting** | annotation으로 요청 제한 |
| **CORS** | annotation으로 헤더 처리 |
| **sticky session** | 같은 사용자를 같은 Pod로 |

> **Spring 앱 입장**: HTTPS 인증서 설정, CORS 필터, rate limit 필터를 코드에서 뺄 수 있다. 앱은 순수 비즈니스 로직만.

---

[← 설정 분리 — ConfigMap / Secret](08-configmap과-secret.md) | [podAntiAffinity와 스케줄링 →](10-podantiaffinity와-스케줄링.md)
