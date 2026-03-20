# SQL — 데이터 분석가의 핵심 도구

> SQL은 도구가 아니라 사고 방식이다. 집합 기반으로 생각하고, 절차적으로 접근하지 말라.

## 핵심 원칙
- SELECT 절은 마지막에 결정한다. FROM → WHERE → GROUP BY → HAVING → SELECT 순으로 사고
- 집합 연산으로 생각하라. 행 단위 루프 사고는 성능과 가독성을 모두 해친다
- NULL은 전염된다. NULL과의 비교는 항상 IS NULL / IS NOT NULL을 사용
- 인덱스를 의식하라. WHERE 절 컬럼에 함수를 씌우면 인덱스가 무력화된다

## 판단 기준
| 상황 | 선택 |
|------|------|
| 단순 집계 | GROUP BY + 집계함수 |
| 누적/이동 평균 | Window Function |
| 계층 구조 탐색 | CTE (재귀) |
| 대용량 중복 제거 | DISTINCT 대신 GROUP BY |
| 조건부 집계 | SUM(CASE WHEN ...) |

## 실전 예시

### Window Function으로 순위 + 누적합 동시에
```sql
SELECT
  user_id,
  order_date,
  amount,
  SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_amount,
  RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS amount_rank
FROM orders;
```

### CTE로 복잡한 로직 단계별 분리
```sql
WITH active_users AS (
  SELECT user_id
  FROM events
  WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY user_id
),
user_revenue AS (
  SELECT user_id, SUM(amount) AS ltv
  FROM orders
  GROUP BY user_id
)
SELECT
  a.user_id,
  COALESCE(r.ltv, 0) AS ltv
FROM active_users a
LEFT JOIN user_revenue r USING (user_id);
```

### 조건부 집계 (Pivot)
```sql
SELECT
  DATE_TRUNC(order_date, MONTH) AS month,
  SUM(CASE WHEN channel = 'organic' THEN amount ELSE 0 END) AS organic_rev,
  SUM(CASE WHEN channel = 'paid' THEN amount ELSE 0 END) AS paid_rev,
  COUNT(DISTINCT user_id) AS buyer_count
FROM orders
GROUP BY 1
ORDER BY 1;
```

## 안티패턴
- `SELECT *` — 컬럼 명시로 의도를 드러내라
- WHERE 절에서 `UPPER(name) = 'KIM'` — 인덱스 무력화, `name = 'kim'` 또는 LOWER 통일
- 서브쿼리 중첩 3단계 이상 — CTE로 분리
- `HAVING COUNT(*) > 0` — 의미 없는 필터, 제거
- JOIN 없이 IN (서브쿼리) 남발 — 대용량에서 EXISTS 또는 JOIN이 빠름

## 실전 팁
- 쿼리 작성 순서: FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
- 디버깅 시 CTE를 하나씩 SELECT해서 중간값 확인
- EXPLAIN / EXPLAIN ANALYZE로 실행계획 반드시 확인
- 날짜 필터는 항상 컬럼 변환 없이: `event_date BETWEEN '2024-01-01' AND '2024-01-31'`
- BigQuery는 파티션 컬럼을 WHERE에 포함시켜 스캔 비용 절감
