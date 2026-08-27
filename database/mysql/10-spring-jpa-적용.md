> [← MySQL 목차](README.md) · [database](../README.md)

# Spring / JPA 환경에서의 적용

## 10.1 격리 수준

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void settle() { ... }
```

- 기본값 `Isolation.DEFAULT`는 **DB 설정을 그대로 따른다**는 의미. MySQL이면 RR.
- 실무에서 격리 수준을 코드로 올리는 일은 드뭅니다. 전역적으로 락 경합과 데드락 위험이 커지기 때문입니다.
- 일반적 접근: **격리 수준은 기본값으로 두고, 문제되는 특정 지점에만 비관적/낙관적 락을 국소 적용.**

## 10.2 인덱스 — JPA에서 놓치기 쉬운 지점

```java
// ✗ JPQL의 함수가 그대로 SQL 함수로 번역됨
@Query("SELECT o FROM Order o WHERE FUNCTION('YEAR', o.orderDate) = :year")

// ✓ 서비스 계층에서 범위를 계산해 넘긴다
LocalDateTime start = LocalDate.of(year, 1, 1).atStartOfDay();
LocalDateTime end   = start.plusYears(1);
List<Order> findByOrderDateGreaterThanEqualAndOrderDateLessThan(
        LocalDateTime start, LocalDateTime end);
```

QueryDSL도 마찬가지로 `order.orderDate.year().eq(2024)` 대신  
`order.orderDate.goe(start).and(order.orderDate.lt(end))`를 씁니다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: debug
```

**실제 생성되는 SQL을 찍어보고 그것을 `EXPLAIN`에 거는 습관**이 중요합니다. ORM이 만든 쿼리라고 최적인 것은 전혀 아닙니다.

## 10.3 외래 키에 대한 실무 관점

외래 키는 정합성을 보장하지만, 부모 테이블에 락이 전파되고 대량 삽입 성능을 떨어뜨립니다. 그래서 대규모 트래픽 서비스나 JPA 프로젝트에서는 **DB 레벨 FK 없이 애플리케이션에서 정합성을 관리**하는 선택도 흔합니다.

```java
@JoinColumn(foreignKey = @ForeignKey(ConstraintMode.NO_CONSTRAINT))
```

다만 이건 트레이드오프이지 "FK를 안 쓰는 게 정답"인 것은 아닙니다.

---

[← EXPLAIN으로 검증하기](09-explain으로-검증하기.md) | [MySQL 핵심 요약 →](11-부록-핵심요약.md)
