# 쿼리 최적화 — 느린 쿼리를 고치는 체계적 접근

> 최적화는 추측이 아니라 측정에서 시작한다. EXPLAIN 없이 튜닝하지 말라.

## 핵심 원칙
- 데이터 스캔량을 줄이는 것이 최우선 (파티션 프루닝, 인덱스 활용)
- 조기 필터링: 가능한 한 이른 단계에서 행을 줄여라
- 집계 전 필터, 조인 전 필터
- 중간 결과셋 크기를 항상 의식하라
- 같은 테이블을 여러 번 스캔하지 말라 (CTE materialization 활용)

## 판단 기준
| 증상 | 원인 | 처방 |
|------|------|------|
| Full table scan | 인덱스 없음/미사용 | WHERE 컬럼 인덱스, 함수 제거 |
| 느린 JOIN | 대용량 Cartesian | JOIN 순서 조정, 필터 먼저 |
| 메모리 초과 | 중간 결과셋 과대 | 서브쿼리 필터 강화 |
| 중복 집계 | 같은 로직 반복 | CTE로 한 번만 계산 |
| 느린 DISTINCT | 정렬 기반 중복제거 | GROUP BY로 대체 |

## 실전 예시

### 나쁜 쿼리 vs 좋은 쿼리
```sql
-- BAD: 전체 테이블 조인 후 필터
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.order_date >= '2024-01-01'
  AND u.country = 'KR';

-- GOOD: 서브쿼리로 먼저 줄인 후 조인
SELECT u.name, o.amount
FROM (
  SELECT user_id, name
  FROM users
  WHERE country = 'KR'
) u
JOIN (
  SELECT user_id, amount
  FROM orders
  WHERE order_date >= '2024-01-01'
) o ON u.user_id = o.user_id;
```

### CTE로 중복 집계 제거
```sql
-- BAD: 같은 집계를 두 번
SELECT
  a.user_id,
  a.total / b.avg_total AS ratio
FROM (SELECT user_id, SUM(amount) AS total FROM orders GROUP BY 1) a
CROSS JOIN (SELECT AVG(total) AS avg_total FROM (SELECT SUM(amount) AS total FROM orders GROUP BY user_id)) b;

-- GOOD: CTE로 한 번 계산
WITH user_totals AS (
  SELECT user_id, SUM(amount) AS total
  FROM orders
  GROUP BY user_id
)
SELECT
  user_id,
  total,
  total / AVG(total) OVER () AS ratio
FROM user_totals;
```

### BigQuery 파티션 활용
```sql
-- BAD: 파티션 컬럼에 함수 적용 → 전체 스캔
SELECT * FROM events
WHERE DATE(event_timestamp) = '2024-01-15';

-- GOOD: 파티션 컬럼 직접 비교
SELECT * FROM events
WHERE event_date = '2024-01-15';  -- DATE 타입 파티션 컬럼
```

## 안티패턴
- WHERE 절에서 `YEAR(date_col) = 2024` — 인덱스/파티션 무력화
- `SELECT COUNT(DISTINCT user_id)` 대용량에서 남발 — HyperLogLog 근사치 고려
- ORDER BY + LIMIT 없이 대용량 결과 반환
- 불필요한 UNION (중복제거 비용) — UNION ALL이 빠름, 중복이 없을 때 사용
- 스칼라 서브쿼리를 SELECT 절에 반복 — JOIN으로 대체

## 실전 팁
- BigQuery: `bq query --dry_run`으로 스캔 바이트 사전 확인
- 실행계획에서 "Hash Join"은 대용량, "Nested Loop"는 소용량에 적합
- Materialized View로 자주 쓰는 집계 결과 캐싱
- 파티션 테이블은 파티션 컬럼을 항상 WHERE에 포함
- APPROX_COUNT_DISTINCT() — BigQuery에서 대용량 unique count 근사치
