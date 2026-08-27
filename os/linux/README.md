# linux — 리눅스와 커널

"먼저 알아야 이해되는 것"부터 배치했습니다. 부팅 → 프로세스 → 파일시스템 → 권한 → 사용자 → 명령어 → 커널 순서입니다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [유닉스 · 리눅스 · POSIX](01-유닉스-리눅스-posix.md) | 유닉스 vs 리눅스, POSIX 표준, 커널 vs 배포판 |
| 02 | [부팅 과정과 커널](02-부팅과-커널.md) | BIOS/UEFI → GRUB → 커널, vmlinuz |
| 03 | [프로세스와 systemd](03-프로세스와-systemd.md) | systemd/systemctl, 데몬, 프로세스 상태, 시그널, 작업 제어 |
| 04 | [파일시스템 구조와 마운트](04-파일시스템과-마운트.md) | FHS, 장치 이름, 마운트, LVM, df/du/free, fstab |
| 05 | [파일 권한](05-파일-권한.md) | rwx, chmod, SetUID/SetGID/Sticky, umask, chown |
| 06 | [사용자와 그룹 관리](06-사용자와-그룹.md) | 기본/보조 그룹, useradd·usermod, sudo vs su |
| 07 | [파일 · 텍스트 명령어](07-파일과-텍스트-명령어.md) | ls/cp/rm/find, grep/tail, 파이프와 리다이렉션, scp vs rsync |
| 08 | [셸 문법 — heredoc과 tee](08-셸-heredoc과-tee.md) | `cat <<EOF`, 파이프, `sudo tee`로 설정 파일 쓰기 |
| 09 | [커널의 역할과 시스템 콜](09-커널의-역할과-시스템콜.md) | 유저/커널 공간 3층 구조, 시스템 콜, 프로세스의 실체 |
| 10 | [CPU 인터럽트](10-cpu-인터럽트.md) | 폴링과의 차이, 컨텍스트 전환 비용, 인터럽트 코얼레싱 |
| 11 | [CFS 스케줄러와 nice](11-cfs-스케줄러와-nice.md) | vruntime, nice 값, cgroup 쿼터와 스로틀링 |
| 12 | [커널 파라미터와 모듈](12-커널-파라미터와-sysctl.md) | sysctl, 모듈 로드, "영구 + 즉시" 2단계 패턴 |
| 99 | [함정과 약자 정리](99-함정과-약자-정리.md) | 데몬↔CLI 패턴, 헷갈리는 함정, 약자 모음 |
