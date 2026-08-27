> [← Linux 목차](README.md) · [os](../README.md)

# 유닉스 · 리눅스 · POSIX

## 유닉스(Unix) vs 리눅스(Linux)
- **유닉스**: 1970년대 벨 연구소에서 만든 원조 OS. 파생된 상용 OS들(Solaris, AIX, HP-UX, macOS의 기반 BSD 등)을 통칭.
- **리눅스**: 리누스 토르발스가 1991년에 유닉스를 "닮게(Unix-like)" 새로 만든 커널. 유닉스 코드를 가져온 게 아니라 철학·인터페이스(POSIX)를 따라 독립 개발.

| 구분 | 유닉스 | 리눅스 |
|---|---|---|
| 라이선스 | 대부분 상용/폐쇄형 | 오픈소스(GPL) |
| 소유 | 특정 회사 | 커널 소스 공개 |
| 예시 | Solaris, AIX, macOS(BSD) | Ubuntu, RHEL 등 |

## POSIX (Portable Operating System Interface)
- 유닉스 계열 OS들이 **공통으로 지켜야 할 표준(규약)**.
- "최소한 이런 함수·명령어·시스템 구조는 이런 이름/방식으로 제공해야 한다"를 규정.
- 규정 범위: 시스템 콜(`fork()`, `wait()`), 명령어(`ls`, `grep`), 쉘 문법, 권한 체계(rwx), 프로세스 개념 등.
- 콘센트 규격 표준에 비유 — 표준을 따르면 서로 호환됨.
- **"문법"보다는 "표준/규약"이 더 정확한 표현.**

## 리눅스 커널 vs 배포판
- **리눅스 = 커널 하나만**을 가리키는 말. 커널은 하드웨어를 제어하고 프로세스/메모리를 관리하는 핵심 부품.
- **배포판(Distribution)** = 커널 + 셸 + 패키지 관리자 + 유틸리티 등을 묶어 "설치해서 쓸 수 있는 완성 OS"로 만든 것.
- 비유: 커널 = 자동차 엔진, 배포판 = 엔진을 넣어 완성한 자동차 브랜드.

**대표 배포판 3계열**

| 계열 | 대표 배포판 | 패키지 관리자 |
|---|---|---|
| Debian 계열 | Ubuntu, Debian | `apt` |
| RHEL 계열 | RHEL, CentOS, Rocky Linux, Fedora | `dnf`/`yum` |
| Arch 계열 | Arch Linux, Manjaro | `pacman` |

- **RHEL** = Red Hat Enterprise Linux(상용). 읽는 법: "알에이치이엘"(가장 무난), "레드햇"(캐주얼).
- **RHEL vs Rocky Linux**: 거의 같은 코드베이스. RHEL은 상용(유료 지원), Rocky는 RHEL 소스를 재빌드한 무료 클론. CentOS의 성격 변화(CentOS Stream 전환) 이후 그 빈자리를 메우려 등장.

---

[부팅 과정과 커널 →](02-부팅과-커널.md)
