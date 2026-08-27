> [← Ansible 목차](README.md) · [devops](../README.md)

# Ansible playbook 작성 원칙

> 한 줄 요약
> **"이 playbook을 지금 다시 돌려도 되는가"를 기준으로 쓴다.**

## ① 멱등성을 가장 먼저 본다

두 번 돌려도 같은 결과가 나와야 **중간에 실패했을 때 처음부터 다시 돌릴 수 있다.**

`shell`이나 `command` 모듈은 멱등이 아니다. 그대로 쓰면 돌릴 때마다 상태가 달라진다.

```yaml
- name: 아카이브 풀기
  ansible.builtin.command: tar xf /tmp/app.tar -C /opt/app
  args:
    creates: /opt/app/bin/run        # 이미 있으면 건너뜀
  # 또는 changed_when 으로 변경 판정을 직접 잡는다
```

## ② 명령형인 것은 상태로 감싼다

명령을 한 번 실행하는 것과, 그 상태가 계속 유지되는 것은 다르다. `iptables` 규칙을 명령으로 넣으면 **재부팅과 함께 사라진다. playbook은 성공했는데 서버는 원상복구되는 상황**이 생긴다.

매번 명령을 때리는 대신 **부팅마다 적용되는 형태**(systemd oneshot 유닛, `sysctl.d` 파일, 설정 파일)로 만들어야 한다.

> **"실행했다"가 아니라 "그 상태가 유지된다"를 목표로 잡는다.**

## ③ 환경 차이는 변수로 빼고, 로직은 복사하지 않는다

환경마다 playbook을 복사하면 **그 순간부터 두 개가 서로 달라지기 시작한다.** 차이는 변수로만 두고 로직은 하나로 유지한다.

```yaml
gpu_enabled: false        # 이 값 하나로 드라이버 설치까지 분기
```

## ④ 순서 의존을 최소화하고 역할 단위로 자른다

한 덩어리로 만들면 **워커 한 대만 추가하고 싶을 때도 전체를 돌려야 한다.** 단계를 나누고 각각 따로 돌릴 수 있게 만든다.

**playbook과 role의 경계**
- **role** = "이 서버가 무엇이 되어야 하는가" (컨테이너 런타임 설치, 공통 보안 설정)
- **playbook** = "어떤 서버들에게 무엇을 시킬 것인가"

이렇게 나눠야 여러 대상이 공통으로 쓰는 부분을 한 번만 작성할 수 있다.

## ⑤ 바꾸는 태스크 옆에 확인하는 태스크를 둔다

설정을 넣는 것과 **그게 실제로 먹었는지 확인하는 것은 다른 일**이다. 확인 태스크가 없으면 **playbook은 성공인데 서비스는 죽어 있는 상태**가 생긴다.

```yaml
- name: 서비스 기동
  ansible.builtin.systemd: { name: nginx, state: started, enabled: true }

- name: 포트가 실제로 열렸는지 확인
  ansible.builtin.wait_for: { port: 80, timeout: 30 }
```

## ⑥ 운영 중인 서버에는 한 번에 치지 않는다

오래된 서버에는 **사람 손을 탄 설정 편차가 반드시 있다.** 그 지도를 먼저 그리지 않고 적용하면 그게 장애가 된다.

```bash
ansible-playbook site.yml --check --diff     # 아무것도 바꾸지 않고 차이만 본다
ansible-playbook site.yml --limit canary     # 한 대에만 먼저
ansible-playbook site.yml --serial 20% --max-fail-percentage 10
```

## 자동화하면 안 되는 것

- **되돌릴 수 없는 작업** — 파티션 재구성, 데이터 삭제. playbook으로 만들면 **실수 한 번이 전 서버에 동시에 적용된다.**
- **판단이 필요한 작업** — 자동화는 **판단이 이미 끝난 일을 반복하는 것**이지 판단 자체를 대신하지 않는다.

---

[← Ansible과 playbook — 동작 원리](01-기초와-동작-원리.md)
