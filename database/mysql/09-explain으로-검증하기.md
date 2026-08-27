> [← MySQL 목차](README.md) · [database](../README.md)

# EXPLAIN으로 검증하기

이론만 알고 끝내면 안 됩니다. 반드시 확인하는 습관을 들여야 합니다.

```sql
EXPLAIN SELECT * FROM orders WHERE YEAR(order_date) = 2024;
```
```
type: ALL          ← 풀 테이블 스캔
key:  NULL         ← 사용된 인덱스 없음
rows: 998234       ← 읽을 것으로 예상되는 행 수
Extra: Using where ← 다 읽고 나서 걸러냄
```

```sql
EXPLAIN SELECT * FROM orders 
 WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```
```
type: range              ← 범위 스캔 ✓
key:  idx_order_date     ← 인덱스 사용 ✓
rows: 51200              ← 읽을 양 대폭 감소
Extra: Using index condition
```

## `type` 읽는 법 (좋은 순서대로)

| type | 의미 |
|---|---|
| `const` / `eq_ref` | PK/UNIQUE로 단 1행 — 최고 |
| `ref` | 인덱스 등호 조건, 여러 행 |
| `range` | **인덱스 범위 스캔** ← 목표 |
| `index` | 인덱스 전체 스캔 (풀스캔보단 낫지만 나쁨) |
| `ALL` | **풀 테이블 스캔** ← 피해야 할 것 |

## `Extra` 읽는 법

| Extra | 의미 |
|---|---|
| `Using index` | **커버링 인덱스** — 테이블 접근조차 안 함 (최고) |
| `Using index condition` | 인덱스 컨디션 푸시다운(ICP) 작동 |
| `Using where` | 읽은 뒤 필터링 |
| `Using filesort` / `Using temporary` | 정렬/임시테이블 발생 — 튜닝 대상 |

> **더 정확한 확인:** `EXPLAIN ANALYZE`(8.0.18+)는 **예상치가 아닌 실제 실행 시간과 실제 읽은 행 수**를 보여줍니다. 옵티마이저의 추정이 틀린 경우를 잡아낼 수 있습니다.

---

[← SARGable — 인덱스를 살리는 조건 작성법](08-sargable-조건-작성법.md) | [Spring / JPA 환경에서의 적용 →](10-spring-jpa-적용.md)
