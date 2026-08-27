> [← kubernetes 목차](README.md)

# 설정 분리 — ConfigMap / Secret

## 문제: 설정을 이미지에 박으면 안 되는 이유

```yaml
# application.yml — jar 안에 그대로 들어감
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/basecamp
    password: mySecret123
```

| 문제 | 상황 |
|---|---|
| ① **환경마다 재빌드** | dev/staging/prod DB가 달라 이미지를 3개 만들어야 함 |
| ② **설정 하나에 재배포** | 로그 레벨만 바꾸는데 빌드→푸시→배포 풀코스 |
| ③ **비밀번호 유출** | 이미지 레이어에 평문. `docker history`로 노출 |

> **원칙**: 이미지는 **어느 환경에서든 똑같아야 한다.** 환경 차이는 **밖에서 주입**한다.
> (12-Factor App 3번 원칙 = Spring profile을 쓰는 이유와 동일)

---

## 11-1. ConfigMap — 설정값 보관함

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:                          # 그냥 키-값 맵
  SPRING_PROFILES_ACTIVE: "prod"
  DB_HOST: "mysql-service"
  LOG_LEVEL: "INFO"
```

```bash
kubectl create configmap my-app-config \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=LOG_LEVEL=INFO

kubectl create configmap my-app-config --from-file=application.yml
```

---

## 11-2. Secret — 비밀번호 보관함 (⚠️ 암호화 아님)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret
type: Opaque
stringData:                    # 평문으로 쓰면 알아서 인코딩됨
  DB_PASSWORD: "mySecret123"
  JWT_SECRET: "abc123xyz"
```

### ★ 반드시 알아야 할 것: Secret은 Base64 인코딩일 뿐이다

```bash
kubectl get secret my-app-secret -o yaml
#   DB_PASSWORD: bXlTZWNyZXQxMjM=      ← 암호처럼 보이지만

echo "bXlTZWNyZXQxMjM=" | base64 -d
# mySecret123                           ← 1초 만에 복호화
```

**Base64는 암호화가 아니라 "바이너리를 텍스트로 표현하는 방식"이다.** 열쇠가 필요 없고 누구나 디코딩할 수 있다.

> **Java 힌트**: `Base64`는 `java.security`가 아니라 `java.util`에 있다. 보안 도구가 아니라 유틸리티라는 뜻.

**그럼 Secret의 존재 의미는?**
- ConfigMap과 역할을 분리해 **RBAC 권한을 따로 부여** 가능
- `kubectl describe` 시 값이 안 찍힘 (실수 방지)
- etcd 저장 시 **암호화를 별도로 켤 수 있음** (EncryptionConfiguration)
- 로그·이벤트에 노출되지 않음

**실무**: AWS Secrets Manager, HashiCorp Vault를 External Secrets Operator로 연동해 진짜 암호화를 한다. k8s Secret만 믿으면 안 된다.

---

## 11-3. Pod에 주입하는 법 2가지

### 방식 A: 환경변수 (가장 많이 씀)

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    envFrom:                        # 통째로 다 넣기
    - configMapRef:
        name: my-app-config
    - secretRef:
        name: my-app-secret
    env:                            # 하나만 골라 넣기
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-app-secret
          key: DB_PASSWORD
```

### ★ Spring Boot Relaxed Binding — 코드 수정 0

Spring Boot는 환경변수를 자동으로 프로퍼티로 변환한다.

```
환경변수  SPRING_DATASOURCE_URL
            ↓ 언더스코어 → 점, 대문자 → 소문자
프로퍼티  spring.datasource.url
```

```yaml
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:mysql://mysql-service:3306/basecamp"
  SPRING_DATASOURCE_USERNAME: "appuser"
  SPRING_JPA_HIBERNATE_DDL_AUTO: "validate"
  LOGGING_LEVEL_ROOT: "INFO"
```

**앱 코드는 한 줄도 안 고쳐도 된다.** `application.yml`이 비어 있어도 위 환경변수만 있으면 DataSource가 붙는다.

**Spring Boot 설정 우선순위:**
```
1. 커맨드라인 인자
2. 환경변수                    ← ConfigMap/Secret이 여기
3. application-{profile}.yml
4. application.yml
```
jar 안에 기본값을 두고 **k8s에서 덮어쓰는** 구조가 자연스럽게 만들어진다.

> `jdbc:mysql://mysql-service:3306` — IP가 아니라 **Service 이름**을 쓴다. CoreDNS가 찾아준다. (→ [서비스 디스커버리](04-서비스디스커버리-service와-coredns.md))

### 방식 B: 볼륨 마운트

```yaml
spec:
  containers:
  - name: my-app
    volumeMounts:
    - name: config-volume
      mountPath: /config          # 이 경로에 파일로 나타남
  volumes:
  - name: config-volume
    configMap:
      name: my-app-config
```

ConfigMap의 **키 = 파일명, 값 = 파일 내용**이 된다.

```yaml
env:
- name: SPRING_CONFIG_ADDITIONAL_LOCATION
  value: "file:/config/"
```

### ★ 선택 기준 — 가장 중요한 함정

| | 환경변수 | 볼륨 마운트 |
|---|---|---|
| 적합 | 짧은 키-값 | 파일 통째로 (yml, 인증서, nginx.conf) |
| **변경 시 반영** | ❌ **Pod 재시작 필요** | ✅ 자동 갱신 (약 1분 지연) |
| 앱에서 읽기 | 자동 (Spring 바인딩) | 파일 읽기 로직 필요 |

**ConfigMap을 수정해도 환경변수로 주입한 Pod는 안 바뀐다.** 환경변수는 프로세스 시작 시 한 번 복사되는 값이기 때문.

```bash
kubectl rollout restart deployment/my-app   # 수정 후 필수
```

> **Java 비유**: `static final`로 초기화 때 값을 복사해둔 것. 원본이 바뀌어도 복사본은 그대로. 볼륨 마운트는 매번 파일을 다시 읽는 것.

---

## 11-4. 전체 그림

```
┌─ ConfigMap ─────────┐   ┌─ Secret ──────────┐
│ SPRING_PROFILES...  │   │ DB_PASSWORD       │
│ SPRING_DATASOURCE_..│   │ JWT_SECRET        │
└──────────┬──────────┘   └────────┬──────────┘
           │      envFrom / valueFrom
           └───────────┬────────────┘
                       ▼
              ┌─────────────────┐
              │  Deployment     │
              └────────┬────────┘
                       ▼
                ┌────┬────┬────┐
                │Pod │Pod │Pod │  ← 환경변수로 설정 주입
                └────┴────┴────┘
```

**환경별 분리:**
```
같은 이미지 my-app:1.0
  ├─ dev 네임스페이스     + ConfigMap(dev)     → dev DB
  ├─ staging 네임스페이스 + ConfigMap(staging) → staging DB
  └─ prod 네임스페이스    + ConfigMap(prod)    → prod DB
```
**이미지는 하나, 설정만 다름.** 이것이 목표였던 그림.

---

[← 클러스터 아키텍처 — 누가 실제로 실행하는가](07-클러스터-아키텍처.md) | [Ingress — 클러스터의 L7 대문 →](09-ingress.md)
