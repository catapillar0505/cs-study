> [← Linux 목차](README.md) · [os](../README.md)

# 셸 문법 — heredoc과 tee

커널 설정 명령어([커널 파라미터와 모듈](12-커널-파라미터와-sysctl.md))를 읽기 위한 사전 문법.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

한 덩어리로 보이지만 3개 부품의 조립이다.

### 부품 ① `cat <<EOF ... EOF` (heredoc)

"지금부터 `EOF`가 나올 때까지 타이핑하는 걸 **한 덩어리 텍스트**로 취급"

```bash
cat <<EOF     ← "EOF 나올 때까지 받아적어"
overlay       ← 내용
br_netfilter  ← 내용
EOF           ← 끝 신호
```

`EOF`는 약속된 이름일 뿐. `cat <<끝` ... `끝` 도 동작한다.

### 부품 ② `|` (파이프)

왼쪽 명령의 **출력** → 오른쪽 명령의 **입력**

### 부품 ③ `sudo tee 파일경로`

받은 내용을 **파일에 저장하면서 화면에도 출력**. (T자 배관처럼 갈라져서 tee)

#### 왜 `sudo echo "..." > 파일` 이 아니라 tee인가? ★

```bash
sudo echo "overlay" > /etc/modules-load.d/k8s.conf
# → Permission denied!
```

**`>` (리다이렉션)은 sudo가 아니라 지금 내 셸이 처리하기 때문.**

```
sudo echo "overlay"  >  /etc/...
└─ 관리자 권한 ─┘   └─ 내 권한(일반 유저) ─┘
```

파일 만드는 담당자(셸)가 일반 유저라 튕긴다.
`tee`는 **자기가 직접 파일을 쓰는 프로그램**이므로 `sudo tee`면 관리자 권한으로 쓸 수 있다.

---

[← 파일 · 텍스트 다루기 명령어](07-파일과-텍스트-명령어.md) | [커널의 역할과 시스템 콜 →](09-커널의-역할과-시스템콜.md)
