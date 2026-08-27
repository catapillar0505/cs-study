> [← kubernetes 목차](README.md)

# kubectl context와 kubeconfig

## 실제 겪은 상황

```
$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop   docker-desktop   docker-desktop
└─ 비어있음!

$ kubectl config use-context docker-desktop
Switched to context "docker-desktop".

$ kubectl config current-context
docker-desktop
```

`CURRENT`가 비어 있으면 `kubectl get pods` 시 이런 에러가 난다:
```
The connection to the server localhost:8080 was refused
```
→ **"어느 클러스터한테 말 걸어야 할지 모르겠다"** 는 뜻. (`localhost:8080`은 설정을 못 찾았을 때 쓰는 하드코딩 기본값)

## kubectl은 사실 아무것도 모른다

**`kubectl`은 클러스터가 아니다. 그냥 HTTP 클라이언트다.**

```
[kubectl] --HTTP 요청--> [API Server] --> [클러스터]
   ↑ 그냥 Postman 같은 놈
```

`kubectl get pods` 의 실체:
```
GET https://192.168.65.3:6443/api/v1/namespaces/default/pods
Authorization: (인증서 또는 토큰)
```

kubectl은 3가지를 **어디선가 알아내야** 한다:
1. **주소** — 어느 서버로? (`https://...:6443`)
2. **신분증** — 나는 누구? (인증서/토큰)
3. **작업 위치** — 어느 네임스페이스?

매번 손으로 치면 지옥이므로 **파일에 적어두고 이름표를 붙인 것**이 context다.

## kubeconfig 구조 (`~/.kube/config`)

```yaml
clusters:              # ① 어디로 (주소록)
- name: docker-desktop
  cluster:
    server: https://127.0.0.1:6443
    certificate-authority-data: LS0tLS1...

users:                 # ② 누구로 (신분증 보관함)
- name: docker-desktop
  user:
    client-certificate-data: LS0tLS1...
    client-key-data: LS0tLS1...

contexts:              # ③ ①+②+네임스페이스 조합에 이름 붙이기
- name: docker-desktop
  context:
    cluster: docker-desktop    # ①에서 고름
    user: docker-desktop       # ②에서 고름
    namespace: default

current-context: docker-desktop   # ← use-context가 바꾸는 게 이 한 줄!
```

## ★ 핵심 공식

```
context = 클러스터(어디로) + 유저(누구로) + 네임스페이스(어느 방에서)
```

`get-contexts` 출력 컬럼이 정확히 이것:

| CURRENT | NAME | CLUSTER | AUTHINFO | NAMESPACE |
|---|---|---|---|---|
| 지금 이거? | 조합 이름 | 어디로 | 누구로 | 어느 방 |

> 셋 다 이름이 `docker-desktop`인 건 **서로 다른 3개 항목이 우연히 이름이 같은 것**뿐. Docker Desktop이 다 같은 이름으로 만들어서 그렇다.

## 왜 전환이 필요한가

실무에선 이렇게 된다:

```
CURRENT   NAME              CLUSTER        NAMESPACE
          docker-desktop    docker-desktop
*         minikube          minikube       dev
          basecamp-prod     eks-prod       production
          basecamp-stage    eks-stage      staging
```

`kubectl delete deployment api-server` 를 쳤을 때:
- context가 `minikube` → 실습 환경 날아감. 괜찮음.
- context가 `basecamp-prod` → **운영 서버 내려감.**

**같은 명령어인데 context에 따라 결과가 완전히 달라진다.** context는 "이 명령이 어디에 떨어질지"를 결정하는 조준경.

> **★ Spring 비유 (가장 정확)**
> `application-dev.yml` / `application-prod.yml` 이 있고, `--spring.profiles.active=prod` 하나로 접속 DB가 바뀌는 것과 완전히 같다.
>
> | Spring | kubectl |
> |---|---|
> | `application.yml` 모음 | `~/.kube/config` |
> | profile (dev/prod) | context |
> | `--spring.profiles.active` | `use-context` |
>
> 잘못된 profile로 띄우면 운영 DB 건드리는 것도 똑같다.

## 명령어 정리

```bash
# 목록 (CURRENT의 * 가 현재)
kubectl config get-contexts

# 현재 것 한 줄로
kubectl config current-context

# 전환
kubectl config use-context docker-desktop

# 설정 전체 보기 (인증서는 REDACTED)
kubectl config view

# 현재 context의 기본 네임스페이스 변경 (자주 씀)
kubectl config set-context --current --namespace=my-app

# 전환 없이 한 번만 다른 context로
kubectl get pods --context=minikube
```

## 사고 방지 습관

```bash
# 1. 위험한 명령 전 확인
kubectl config current-context && kubectl delete deployment xxx

# 2. 프롬프트에 context 띄우기 (kube-ps1)
(⎈|basecamp-prod:production) user@DESKTOP:~$

# 3. 편의 도구
kubectx              # context 목록 + 선택
kubens my-app        # 네임스페이스 빠른 전환
alias k=kubectl      # 오타 고통 반감
```

---

[← 네트워크 — 오버레이와 언더레이](02-네트워크-오버레이와-언더레이.md) | [서비스 디스커버리 — Service + CoreDNS →](04-서비스디스커버리-service와-coredns.md)
