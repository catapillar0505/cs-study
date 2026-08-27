> [← MySQL 목차](README.md) · [database](../README.md)

# SARGable — 인덱스를 살리는 조건 작성법

## 8.1 문제 상황

```sql
-- (A) 인덱스 무력화
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- (B) 인덱스 활용
SELECT * FROM orders 
 WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```

**결과는 완전히 동일**한데 성능은 수십~수백 배 차이가 납니다.

## 8.2 이유 — 인덱스에 저장된 것은 "원본 값"이다

```
인덱스에 저장된 값:  2024-01-03, 2024-01-07, 2024-02-01, ...
쿼리가 묻는 값:      YEAR(...) = 2024
```

**인덱스 어디에도 `YEAR(order_date)`의 결과값은 저장되어 있지 않습니다.** 그래서 DB는 이렇게 할 수밖에 없습니다.

```
행 1: order_date = 2024-01-03 → YEAR() 계산 → 2024 → 일치 ✓
행 2: order_date = 2023-11-20 → YEAR() 계산 → 2023 → 불일치 ✗
...
행 1000000: ...
```

**모든 행을 꺼내 함수를 적용해봐야 합니다.** 이것이 풀 테이블 스캔입니다.

## 8.3 더 근본적으로 — 정렬이 깨진다

함수를 통과한 값의 순서는 원본의 순서와 무관해질 수 있습니다.

```
원본 정렬:      1, 2, 3, 4, 5
MOD(x, 3):      1, 2, 0, 1, 2      -- 순서 완전 파괴 ★
```

**옵티마이저는 함수의 수학적 성질을 하나하나 알지 못합니다.** 그래서 **"인덱스 컬럼에 함수가 씌워지면 그 인덱스는 못 쓴다"** 고 보수적으로 판단합니다.

> **원칙 한 줄**  
> **인덱스 컬럼은 비교 연산자의 왼쪽에 홀로, 아무 가공 없이 두어라.  
> 계산이 필요하면 반대편(상수 쪽)에서 하라.**

## 8.4 SARGable — 정식 용어

> **SARGable = Search ARGument able**  
> "검색 인자로 쓸 수 있는", 즉 **인덱스 탐색에 직접 사용 가능한 조건**

인덱스가 있어도 조건은 두 가지 역할로 갈립니다.

| | Access 조건 | Filter 조건 |
|---|---|---|
| 역할 | **읽을 범위를 결정** | 읽어온 행을 **걸러냄** |
| 성능 영향 | 읽는 양 자체를 줄임 ★ | 이미 다 읽은 뒤 버림 |
| EXPLAIN | `type: range/ref` | `Extra: Using where` |

**함수를 씌운 조건은 Access 조건이 되지 못하고 Filter 조건으로 강등됩니다.** 결과는 맞지만 읽는 양이 줄지 않습니다.

## 8.5 Non-SARGable 패턴 모음

### (1) 날짜 함수 — 가장 흔한 케이스

```sql
-- ✗ 인덱스 무력화
WHERE YEAR(order_date) = 2024
WHERE DATE(order_date) = '2024-05-01'
WHERE DATE_FORMAT(order_date, '%Y-%m') = '2024-05'
WHERE order_date + INTERVAL 1 DAY > NOW()

-- ✓ 범위 조건으로 변환
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
WHERE order_date >= '2024-05-01' AND order_date < '2024-05-02'
WHERE order_date >= '2024-05-01' AND order_date < '2024-06-01'
WHERE order_date > NOW() - INTERVAL 1 DAY
```

마지막 예시는 **`+ INTERVAL 1 DAY`를 컬럼 쪽에서 상수 쪽으로 이항**한 것입니다. SARGable 변환의 전형입니다.

> **경계값 주의 — `BETWEEN`**
> ```sql
> -- ✗ 위험 (order_date가 DATETIME인 경우)
> WHERE order_date BETWEEN '2024-05-01' AND '2024-05-31'
> ```
> `'2024-05-31'`은 `'2024-05-31 00:00:00'`으로 해석되어 **5월 31일 00:00:01 이후 데이터가 전부 누락됩니다.**
> ```sql
> -- ✓ 안전: 시작은 이상, 끝은 미만
> WHERE order_date >= '2024-05-01' AND order_date < '2024-06-01'
> ```
> `>= 시작 AND < 다음구간시작` 패턴이면 DATE/DATETIME/TIMESTAMP 어느 타입이든 안전합니다.

### (2) 산술 연산

```sql
-- ✗                          -- ✓
WHERE price * 1.1 > 11000     WHERE price > 10000
WHERE salary / 12 > 5000      WHERE salary > 60000
```

### (3) 문자열 함수와 LIKE

```sql
-- ✗                                        -- ✓
WHERE SUBSTRING(phone, 1, 3) = '010'        WHERE phone LIKE '010%'
WHERE UPPER(email) = 'TEST@EXAMPLE.COM'     WHERE email = 'test@example.com'
```

**`LIKE`는 앞부분이 고정되어야만 인덱스를 씁니다.**

```sql
WHERE name LIKE '김%'     -- ✓ range 스캔 가능
WHERE name LIKE '%김'     -- ✗ 풀스캔
WHERE name LIKE '%김%'    -- ✗ 풀스캔
```

인덱스는 **왼쪽부터** 정렬되어 있습니다. `'김'`으로 시작하는 값은 연속 구간을 이루지만, `'김'`으로 **끝나는** 값은 인덱스 전체에 흩어져 있습니다.

> 뒷부분 검색이 꼭 필요하면 → **FULLTEXT 인덱스**, 또는 문자열을 뒤집어 저장한 컬럼에 인덱스.

### (4) 암묵적 형변환 — 가장 발견하기 어려운 함정

```sql
-- phone 컬럼이 VARCHAR인 경우
WHERE phone = 01012345678     -- ✗ 컬럼이 숫자로 변환됨 → 인덱스 무력화
WHERE phone = '01012345678'   -- ✓
```

> **MySQL의 비교 규칙:** 문자열과 숫자를 비교하면 **문자열을 숫자로** 변환합니다. 즉 컬럼이 문자열이면 **컬럼이 변환당합니다.** 반대로 컬럼이 INT인데 `WHERE id = '123'`이면 상수 쪽이 변환되므로 인덱스가 살아 있습니다.

**JOIN에서도 동일하게 발생합니다.** 조인 키의 타입이 다르거나(`VARCHAR` vs `INT`), **콜레이션이 다르면**(`utf8mb4_general_ci` vs `utf8mb4_unicode_ci`) 형변환으로 인덱스가 무력화됩니다. 조인이 갑자기 느려질 때 반드시 확인할 항목입니다.

### (5) 부정 조건과 OR

```sql
WHERE status != 'DONE'
WHERE status NOT IN ('A', 'B')
WHERE user_id = 1 OR order_no = 'X'   -- 각 컬럼에 인덱스 없으면 풀스캔
```

인덱스는 "여기부터 여기까지"를 좁히는 도구인데, `!=`는 사실상 전체를 가리킵니다.  
`OR`은 index merge로 처리될 수도 있지만 옵티마이저가 풀스캔을 고르는 경우가 많아, `UNION ALL` 분리가 더 빠를 때가 있습니다.

### (6) 복합 인덱스의 후행 컬럼 가공

```sql
INDEX idx_user_date (user_id, order_date)

WHERE user_id = 1 AND YEAR(order_date) = 2024
```

`user_id = 1`까지는 인덱스를 타지만 **`order_date`는 Filter로만 작동**합니다. 그 사용자의 주문이 10만 건이면 10만 건을 다 읽습니다.

> **복합 인덱스는 앞 컬럼부터 등호 조건이 이어질 때 가장 효율적**이고, **범위 조건이 나오는 순간 그 뒤 컬럼은 Access 조건으로 쓰이지 못합니다.**

## 8.6 심화 — 그래도 함수를 써야 한다면

### 함수 기반 인덱스 (MySQL 8.0.13+)

```sql
CREATE INDEX idx_order_year ON orders ((YEAR(order_date)));
--                                     ↑↑ 괄호 두 겹 필수

SELECT * FROM orders WHERE YEAR(order_date) = 2024;   -- 이제 인덱스 사용 ✓
```

**`YEAR()`의 결과값으로 정렬된 별도 인덱스를 만든 것**입니다. 8.2의 문제를 정면으로 해결한 방식이죠.

주의: 쿼리의 표현식이 인덱스 정의와 **정확히 일치**해야 합니다.

### 생성 컬럼 (Generated Column)

```sql
ALTER TABLE orders 
  ADD COLUMN order_year INT AS (YEAR(order_date)) STORED,
  ADD INDEX idx_order_year (order_year);
```

- `STORED`: 디스크에 실제 저장 (공간 소모)
- `VIRTUAL`: 읽을 때 계산 (8.0에서는 이것도 인덱스 가능)

### 그런데 대부분의 경우 함수 인덱스는 불필요합니다

`order_date` 인덱스 하나면 **연도, 월, 일, 임의 기간을 전부 커버**합니다.

```
idx_order_date 하나로:
  2024년 전체 ✓ / 2024년 5월 ✓ / 최근 7일 ✓ / 특정 2주 구간 ✓
```

반면 `YEAR()` 함수 인덱스는 **연도 조회에만** 쓰입니다. **인덱스는 공짜가 아닙니다** — 공간을 차지하고 INSERT/UPDATE/DELETE마다 갱신되어 쓰기 성능을 떨어뜨립니다.

> **원칙: 쿼리를 SARGable하게 고칠 수 있으면 그것이 우선이고, 함수 인덱스는 도저히 쿼리를 바꿀 수 없을 때(레거시, ORM 제약)의 차선책입니다.**

## 8.7 반대 방향의 오해 — "인덱스를 쓰면 항상 빠르다"가 아니다

**옵티마이저는 인덱스를 쓸 수 있어도 일부러 안 쓰기도 합니다.**

```sql
WHERE order_date >= '2020-01-01'   -- 전체의 95%가 해당
```

인덱스를 타면:

```
1. 인덱스에서 PK를 얻고
2. PK로 클러스터형 인덱스를 다시 타서 실제 행을 읽고   ← 랜덤 I/O
3. 이걸 95만 번 반복
```

**차라리 테이블을 순차적으로 쭉 읽는 게 빠릅니다.** 일반적으로 **조회 대상이 전체의 20~25%를 넘으면 옵티마이저는 풀스캔을 선택**합니다.

이 판단은 **통계 정보(카디널리티)** 기반이며, 통계가 낡으면 잘못된 선택을 합니다.

```sql
ANALYZE TABLE orders;   -- 통계 정보 갱신
```

> **커버링 인덱스:** 위 2번의 "테이블 재접근"이 병목입니다. 필요한 컬럼이 인덱스 안에 다 있으면 2번을 통째로 생략할 수 있어 극적으로 빨라집니다. 실행 계획에 `Using index`로 표시되며, **`SELECT *`를 피하라는 조언의 실질적 근거**가 이것입니다.

---

[← 인덱스의 원리](07-인덱스의-원리.md) | [EXPLAIN으로 검증하기 →](09-explain으로-검증하기.md)
