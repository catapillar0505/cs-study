> [← MySQL 목차](README.md) · [database](../README.md)

# InnoDB의 구현 — MVCC · 락 · 로그

## 5.1 두 갈래의 구현 방식

**방식 A — 락 기반 (전통적)**

읽을 때 공유 락(S), 쓸 때 배타 락(X)을 걸고 트랜잭션 종료까지 유지(2PL).

- READ COMMITTED → 읽기 락을 읽는 순간만 잡고 놓음
- REPEATABLE READ → 읽기 락을 끝까지 유지

**치명적 단점: 읽기와 쓰기가 서로를 막습니다.** 조회가 압도적으로 많은 웹 서비스에서 이는 재앙입니다.

**방식 B — MVCC (현대 DB의 답)**

> **데이터를 덮어쓰지 말고, 옛날 버전을 남겨두자.**  
> 쓰는 사람은 새 버전을 만들고, 읽는 사람은 옛날 버전을 읽는다.  
> → **읽기는 쓰기를 막지 않고, 쓰기는 읽기를 막지 않는다.**

MySQL InnoDB, PostgreSQL, Oracle이 모두 채택했습니다.

## 5.2 MVCC의 구성 요소

**① 언두 로그 (Undo Log)**

`UPDATE` 실행 시 InnoDB는  
1. 변경 **전** 데이터를 언두 로그에 복사하고  
2. 실제 데이터 페이지를 새 값으로 수정합니다.

언두 로그는 원래 **롤백**을 위한 장치였는데, 이 "과거 버전 저장소"를 **MVCC의 읽기 소스로 재활용**한 것입니다.

**② Read View (스냅샷)**

SELECT 시 "그 시점에 어떤 트랜잭션이 활성 상태였는가"를 기록한 Read View를 만듭니다. 데이터를 읽을 때 각 행의 버전을 보고 판단합니다.

- Read View 생성 이전에 커밋된 버전 → 읽어도 됨
- 아직 활성 중이거나 나중에 커밋됨 → **언두 로그를 타고 더 과거 버전으로 이동**

## 5.3 RC와 RR의 진짜 차이는 단 한 줄

| | Read View를 언제 만드는가 |
|---|---|
| `READ COMMITTED` | **SELECT 할 때마다 새로** |
| `REPEATABLE READ` | **트랜잭션의 첫 SELECT 때 한 번만** |

- RC는 매번 새 스냅샷 → 그 사이 커밋된 변경이 보임 → **Non-Repeatable Read 발생**
- RR은 스냅샷 고정 → 트랜잭션 내내 같은 시점의 세계 → **Non-Repeatable Read 차단**

그리고 중요한 결과가 하나 더 나옵니다.

**RR의 고정된 스냅샷은 INSERT된 행도 걸러냅니다.** 즉 **InnoDB는 RR에서 락 없이도 Phantom Read를 상당 부분 막습니다.** (표준 이론과 달라지는 지점)

## 5.4 Consistent Read vs Current Read

MVCC 스냅샷은 **일반 SELECT**에만 적용됩니다.

| 구분 | 대상 | 읽는 데이터 |
|---|---|---|
| **Consistent Read** | 일반 `SELECT` | 스냅샷 (과거 시점) |
| **Current Read** | `SELECT ... FOR UPDATE`, `LOCK IN SHARE MODE`, `UPDATE`, `DELETE` | **최신 데이터 + 락** |

당연합니다. 5초 전 스냅샷을 보고 UPDATE하면 남의 변경을 덮어쓰게 되니까요.

## 5.5 Gap Lock과 Next-Key Lock

Current Read에서는 락으로 팬텀을 막습니다.

```
Record Lock : 실제 존재하는 행 자체를 잠금
Gap Lock    : 행과 행 "사이의 빈 공간"을 잠금 → 여기에 INSERT 불가
Next-Key Lock = Record Lock + Gap Lock
```

인덱스에 `age` 값이 `10, 20, 30`으로 있고 `WHERE age BETWEEN 15 AND 25 FOR UPDATE`를 하면, 20이라는 행뿐 아니라 **10~20 사이, 20~30 사이의 빈 공간까지** 잠급니다. 다른 트랜잭션이 `age = 22`를 INSERT하려 하면 대기합니다.

> **실무 주의:** Gap Lock은 존재하지 않는 데이터까지 잠그므로 **데드락과 락 대기의 주범**입니다. 인덱스가 없는 컬럼으로 조건을 걸면 풀스캔하며 사실상 테이블 전체에 갭 락이 걸립니다. `FOR UPDATE`를 쓸 때 인덱스 설계가 중요한 이유입니다.

## 5.6 그래도 남는 팬텀 케이스

스냅샷 읽기와 잠금 읽기가 **섞일 때** InnoDB RR에서도 팬텀이 보입니다.

```
T1: SELECT * FROM member WHERE age = 25;              -- 0건 (스냅샷 생성)
                T2: INSERT INTO member VALUES (100, 25); COMMIT;
T1: SELECT * FROM member WHERE age = 25;              -- 0건 (스냅샷 유지)
T1: SELECT * FROM member WHERE age = 25 FOR UPDATE;   -- 1건 ★ 팬텀
```

세 번째 쿼리는 Current Read라 최신 데이터를 보기 때문입니다. T1이 그 행을 직접 `UPDATE`해도 그 순간부터 자기 스냅샷에 보이기 시작합니다.

> **정확한 서술:** InnoDB는 RR에서 팬텀을 **대부분 방지**하지만, **잠금 읽기가 섞이면 예외적으로 발생**할 수 있다.

## 5.7 Redo Log와 Undo Log

### 배경 — 왜 로그가 필요한가

InnoDB는 **버퍼 풀(Buffer Pool)** 이라는 메모리에 데이터를 올려놓고 작업합니다.

```
UPDATE 실행 시:
1. 디스크에서 페이지를 버퍼 풀로 읽어옴
2. 버퍼 풀 메모리 상에서 값을 수정   ← 이 시점엔 디스크는 옛날 값
3. 나중에 한가할 때 디스크에 반영 (flush)
```

수정된 채 아직 디스크에 안 내려간 페이지를 **더티 페이지(Dirty Page)** 라고 합니다. 2와 3 사이에 서버가 죽으면 커밋된 변경이 사라집니다(지속성 위반). 그렇다고 커밋마다 데이터 페이지를 쓰자니, 페이지가 디스크 여기저기 흩어져 있어 **랜덤 I/O**라 너무 느립니다.

### Redo Log — WAL (Write-Ahead Logging)

**해법: 무거운 데이터 페이지 대신 가벼운 "변경 기록"만 순차 쓰기로 먼저 저장한다.**

```
커밋 시점:
  데이터 페이지 → 디스크 (X, 랜덤 I/O라 느림, 나중에)
  Redo Log      → 디스크 (O, 순차 I/O라 빠름, 즉시)
```

서버가 죽어도 재시작 시 Redo Log를 읽어 **아직 반영 안 된 커밋된 변경을 다시 실행(REDO)** 하면 복구됩니다.

```sql
innodb_flush_log_at_trx_commit = 1  -- 커밋마다 fsync (기본, 완전 안전)
                               = 2  -- OS 캐시까지, 1초마다 fsync (DB 죽어도 OK)
                               = 0  -- 1초마다 (가장 빠르나 최대 1초 유실)
```

### 비교

| | Redo Log | Undo Log |
|---|---|---|
| 저장 내용 | 변경 **후** 정보 (어떻게 다시 할지) | 변경 **전** 정보 (어떻게 되돌릴지) |
| 방향 | 재실행 (앞으로) | 취소 (뒤로) |
| ACID 담당 | **D**urability | **A**tomicity |
| 부가 용도 | 크래시 복구 | **MVCC 스냅샷 읽기** |
| 정리 시점 | 체크포인트 이후 재사용 | 참조 트랜잭션이 모두 끝난 후 |

### Redo Log ≠ Binary Log (자주 혼동)

| | Redo Log | Binary Log (binlog) |
|---|---|---|
| 소속 계층 | **InnoDB (스토리지 엔진)** | **MySQL 엔진 (서버)** |
| 기록 내용 | 페이지 물리적 변경 | 논리적 변경 (SQL 또는 행 이미지) |
| 목적 | **크래시 복구** | **복제(Replication), 시점 복구(PITR)** |
| 크기 | 고정, 순환 재사용 | 계속 누적 |

둘의 순서를 맞추기 위해 커밋 시 내부적으로 **2단계 커밋(2PC)** 이 수행됩니다.

## 5.8 InnoDB의 다른 핵심 구조

### 클러스터형 인덱스 (Clustered Index)

InnoDB에서 **테이블은 곧 PK 기준으로 정렬된 B+Tree**입니다. 데이터가 인덱스 리프 노드 안에 직접 들어있습니다.

```
InnoDB (클러스터형)              MyISAM (비클러스터형)

PK 인덱스 B+Tree                 인덱스              데이터 파일
  └─ 리프에 실제 행 데이터          └─ 리프에 행 주소 → [.MYD 파일의 행]
```

- PK 조회가 매우 빠름
- **세컨더리 인덱스는 리프에 PK 값을 저장** → 세컨더리로 조회하면 PK를 얻고 **다시 PK 인덱스를 타야** 함
- **PK가 무작위 값(UUID)이면** B+Tree 중간에 끼워 넣게 되어 페이지 분할이 잦음 → **AUTO_INCREMENT나 순차 값 권장**

### 더블 라이트 버퍼 (Doublewrite Buffer)

Redo Log만으로 부족한 케이스가 있습니다. **부분 페이지 쓰기(Torn Page)** 문제입니다.

InnoDB 페이지는 16KB인데 OS/디스크는 보통 4KB 단위로 씁니다. 16KB 중 8KB만 쓰인 상태에서 전원이 나가면 **페이지 자체가 깨집니다.** Redo Log는 "이 페이지에 이 변경을 적용하라"는 내용인데, 페이지가 깨졌으면 적용할 기반이 없습니다.

그래서 InnoDB는 데이터 페이지를 실제 위치에 쓰기 전에 **별도 영역에 통째로 한 번 먼저 씁니다.** 크래시 후엔 이 사본으로 복원한 뒤 Redo를 적용합니다. **MyISAM에는 이런 장치가 전혀 없습니다.**

### 버퍼 풀

- InnoDB: **데이터와 인덱스를 모두** 캐싱
- MyISAM: **인덱스만**(`key_buffer`) 캐싱, 데이터는 OS 캐시에 의존

```sql
innodb_buffer_pool_size = 물리 메모리의 50~70%   -- 가장 중요한 튜닝 파라미터
```

---

[← 트랜잭션 격리 수준 4단계](04-트랜잭션-격리수준.md) | [표준에 없는 이상 현상 — Lost Update · Write Skew →](06-표준밖-이상현상.md)
