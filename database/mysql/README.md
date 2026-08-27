# mysql — MySQL 내부 동작

스토리지 엔진 → 트랜잭션 → 동시성과 격리 수준 → 인덱스 순서로, 물리적 토대에서 논리적 개념으로 올라갑니다.

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [아키텍처와 스토리지 엔진](01-아키텍처와-스토리지엔진.md) | 2계층 구조, 엔진별 차이, 락 단위 |
| 02 | [트랜잭션과 ACID](02-트랜잭션과-acid.md) | 커밋 전 중간 상태, ACID와 구현 수단 |
| 03 | [동시성 문제와 이상 현상](03-동시성-문제와-이상현상.md) | Dirty / Non-Repeatable / Phantom Read |
| 04 | [트랜잭션 격리 수준](04-트랜잭션-격리수준.md) | 4단계, DBMS별 기본값, MySQL만 RR인 이유 |
| 05 | [InnoDB의 구현](05-innodb-mvcc와-락.md) | 언두 로그, Read View, Gap/Next-Key Lock, Redo vs Binlog |
| 06 | [표준에 없는 이상 현상](06-표준밖-이상현상.md) | Lost Update, Write Skew와 대응책 |
| 07 | [인덱스의 원리](07-인덱스의-원리.md) | B+Tree, 리프 연결과 범위 스캔 |
| 08 | [SARGable 조건 작성법](08-sargable-조건-작성법.md) | 함수가 인덱스를 죽이는 이유, 함수 인덱스 |
| 09 | [EXPLAIN으로 검증하기](09-explain으로-검증하기.md) | type / Extra 읽는 법 |
| 10 | [Spring / JPA 적용](10-spring-jpa-적용.md) | 격리 수준 설정, JPQL 함수 함정, FK 실무 관점 |
| 11 | [핵심 요약](11-부록-핵심요약.md) | 전체 흐름, 한 줄 원칙, 자주 나오는 함정 |
