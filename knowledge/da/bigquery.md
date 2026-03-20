# BigQuery — 구글 클라우드 데이터 웨어하우스 실전 가이드

> BigQuery에서 비용은 스캔 바이트로 결정된다. 파티션과 클러스터링이 비용 절감의 핵심이다.

## 핵심 원칙
- 파티션 컬럼을 WHERE에 항상 포함하라 (비용 = 스캔 바이트)
- SELECT * 대신 필요한 컬럼만 선택하라
- 클러스터링: 자주 필터링하는 컬럼 기준 (최대 4개)
- Materialized View로 자주 쓰는 집계 캐싱
- dry_run으로 쿼리 비용 사전 확인

## 판단 기준
| 기능 | 사용 시점 |
|------|-----------|
| 파티션 | 날짜 기반 필터링이 있는 대용량 테이블 |
| 클러스터링 | 특정 컬럼(user_id, country 등) 필터가 잦은 경우 |
| Materialized View | 일별/주별 집계를 여러 쿼리가 반복 사용 |
| BI Engine | Looker Studio 연동 시 대화형 쿼리 캐싱 |
| BigQuery ML | SQL로 ML 모델 빠르게 빌드 |

## 실전 예시

### 파티션 + 클러스터링 테이블 생성
```sql
CREATE TABLE `project.dataset.events`
(
  event_id STRING NOT NULL,
  user_id STRING NOT NULL,
  event_name STRING,
  event_date DATE NOT NULL,           -- 파티션 키
  event_timestamp TIMESTAMP,
  properties JSON
)
PARTITION BY event_date
CLUSTER BY user_id, event_name        -- 자주 쓰는 필터 컬럼
OPTIONS (
  partition_expiration_days = 365,    -- 1년 후 자동 삭제
  require_partition_filter = TRUE     -- 파티션 필터 없으면 에러
);
```

### BigQuery 특화 함수 활용
```sql
-- ARRAY_AGG: 행을 배열로 집계
SELECT
  user_id,
  ARRAY_AGG(
    STRUCT(event_name, event_timestamp)
    ORDER BY event_timestamp
    LIMIT 10
  ) AS recent_events
FROM events
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY user_id;

-- JSON 컬럼 파싱
SELECT
  user_id,
  JSON_VALUE(properties, '$.page_url') AS page_url,
  CAST(JSON_VALUE(properties, '$.duration_sec') AS INT64) AS duration_sec
FROM events
WHERE event_name = 'page_view'
  AND event_date = CURRENT_DATE() - 1;

-- APPROX_COUNT_DISTINCT: 대용량 unique count 근사 (빠름)
SELECT
  event_date,
  APPROX_COUNT_DISTINCT(user_id) AS approx_dau
FROM events
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY event_date
ORDER BY event_date;
```

### BigQuery ML 예시
```sql
-- 이탈 예측 모델 (SQL만으로)
CREATE OR REPLACE MODEL `project.dataset.churn_model`
OPTIONS(
  model_type = 'LOGISTIC_REG',
  input_label_cols = ['churned'],
  auto_class_weights = TRUE
) AS
SELECT
  recency_days,
  frequency,
  total_revenue,
  feature_breadth,
  days_since_first_order,
  churned  -- 레이블
FROM `project.dataset.churn_features`
WHERE split = 'TRAIN';

-- 예측 실행
SELECT
  user_id,
  predicted_churned,
  predicted_churned_probs[OFFSET(1)].prob AS churn_probability
FROM ML.PREDICT(
  MODEL `project.dataset.churn_model`,
  (SELECT * FROM `project.dataset.churn_features` WHERE split = 'PREDICT')
)
ORDER BY churn_probability DESC
LIMIT 1000;
```

### Python bigquery 클라이언트 실전
```python
from google.cloud import bigquery
import pandas as pd

client = bigquery.Client(project='your-project')

def run_query(sql: str, params: dict = None) -> pd.DataFrame:
    """파라미터 쿼리 실행 (SQL Injection 방지)"""
    job_config = bigquery.QueryJobConfig()

    if params:
        job_config.query_parameters = [
            bigquery.ScalarQueryParameter(k, 'STRING', v)
            for k, v in params.items()
        ]

    job = client.query(sql, job_config=job_config)
    return job.to_dataframe()

# 비용 사전 확인
def estimate_query_cost(sql: str) -> float:
    job_config = bigquery.QueryJobConfig(dry_run=True)
    job = client.query(sql, job_config=job_config)
    bytes_processed = job.total_bytes_processed
    cost_usd = bytes_processed / 1e12 * 6.25  # $6.25 per TB
    print(f"예상 스캔: {bytes_processed/1e9:.2f} GB")
    print(f"예상 비용: ${cost_usd:.4f}")
    return cost_usd
```

## 안티패턴
- `SELECT *` + 대용량 테이블 — 불필요한 비용
- 파티션 필터 없는 쿼리 — 전체 테이블 스캔
- Repeated CROSS JOIN — 예상치 못한 폭발적 비용
- 스트리밍 INSERT를 자주 사용 — 배치 적재가 비용 효율적
- 비파티션 테이블에 대용량 데이터 누적

## 실전 팁
- 비용 알림: $100/일 초과 시 Slack 알림 설정
- Slot 예약: 대용량 워크로드는 온디맨드보다 예약이 저렴
- 테이블 만료 설정: 임시 테이블은 자동 삭제 (7일)
- Information Schema: 쿼리 비용/성능 모니터링
  ```sql
  SELECT user_email, SUM(total_bytes_processed)/1e12 AS tb_processed
  FROM `region-asia-northeast3`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE DATE(creation_time) = CURRENT_DATE()
  GROUP BY 1 ORDER BY 2 DESC;
  ```
