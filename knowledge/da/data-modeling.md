# 데이터 모델링 — 분석하기 좋은 데이터 구조 설계

> 좋은 데이터 모델은 분석가가 SQL을 덜 쓰게 만든다.

## 핵심 원칙
- 분석 목적에 맞게 모델링하라 (OLTP와 OLAP는 다르다)
- Denormalization: 분석용 DW는 조인을 줄이기 위해 비정규화 허용
- 일관된 명명 규칙: 누가 봐도 의미를 알 수 있는 이름
- 느린 변경 차원(SCD)을 설계에 반영하라
- 파티션과 클러스터링을 처음부터 계획하라

## 판단 기준
| 모델 유형 | 구조 | 적합한 용도 |
|-----------|------|-------------|
| Star Schema | 팩트 + 차원 테이블 | 단순 집계, BI 도구 연동 |
| Snowflake Schema | 차원 테이블 정규화 | 데이터 일관성 중요할 때 |
| One Big Table (OBT) | 완전 비정규화 | 빠른 쿼리, 단순한 분석 |
| Data Vault | Hub-Satellite-Link | 변경 이력 추적, 대규모 DW |

## 실전 예시

### Star Schema 설계 예시
```sql
-- 팩트 테이블 (fact_orders)
CREATE TABLE fact_orders (
  order_id STRING NOT NULL,
  user_id STRING NOT NULL,          -- FK to dim_users
  product_id STRING NOT NULL,       -- FK to dim_products
  date_id DATE NOT NULL,            -- FK to dim_date, 파티션 키
  channel_id STRING NOT NULL,       -- FK to dim_channels

  -- 측정값
  quantity INT64,
  unit_price NUMERIC,
  discount_amount NUMERIC,
  gross_revenue NUMERIC,
  net_revenue NUMERIC,

  -- 메타데이터
  created_at TIMESTAMP,
  updated_at TIMESTAMP
)
PARTITION BY date_id
CLUSTER BY user_id, product_id;

-- 차원 테이블 (dim_users)
CREATE TABLE dim_users (
  user_id STRING NOT NULL,
  user_name STRING,
  email STRING,
  country STRING,
  user_segment STRING,       -- 'enterprise', 'smb', 'individual'
  acquisition_channel STRING,
  first_order_date DATE,

  -- SCD Type 2 컬럼
  valid_from TIMESTAMP,
  valid_to TIMESTAMP,
  is_current BOOLEAN
);
```

### dbt 모델 레이어 구조
```
models/
├── staging/          # 소스 데이터 그대로 + 타입 변환
│   ├── stg_orders.sql
│   └── stg_users.sql
├── intermediate/     # 비즈니스 로직 중간 단계
│   ├── int_user_metrics.sql
│   └── int_order_aggregates.sql
└── marts/           # 최종 분석용 테이블
    ├── fct_orders.sql
    ├── dim_users.sql
    └── wide_user_summary.sql
```

```sql
-- dbt 모델 예시: stg_orders.sql
WITH source AS (
  SELECT * FROM {{ source('raw', 'orders') }}
),
renamed AS (
  SELECT
    CAST(id AS STRING)           AS order_id,
    CAST(user_id AS STRING)      AS user_id,
    CAST(amount AS NUMERIC)      AS gross_revenue,
    CAST(discount AS NUMERIC)    AS discount_amount,
    CAST(amount - discount AS NUMERIC) AS net_revenue,
    DATE(created_at)             AS order_date,
    TIMESTAMP(created_at)        AS created_at
  FROM source
  WHERE id IS NOT NULL
    AND amount >= 0
)
SELECT * FROM renamed
```

### Wide Table (One Big Table) 패턴
```sql
-- 사용자별 통합 요약 테이블
CREATE OR REPLACE TABLE mart_user_summary AS
SELECT
  u.user_id,
  u.acquisition_channel,
  u.country,

  -- 행동 지표
  e.total_sessions,
  e.last_active_date,
  e.days_since_last_active,
  e.feature_breadth,

  -- 구매 지표
  o.first_order_date,
  o.last_order_date,
  o.total_orders,
  o.total_revenue,
  o.avg_order_value,

  -- 세그먼트
  CASE
    WHEN o.total_revenue >= 1000000 THEN 'high_value'
    WHEN o.total_revenue >= 100000  THEN 'mid_value'
    WHEN o.total_revenue > 0        THEN 'low_value'
    ELSE 'non_purchaser'
  END AS value_segment

FROM dim_users u
LEFT JOIN int_user_event_metrics e USING (user_id)
LEFT JOIN int_user_order_metrics o USING (user_id)
WHERE u.is_current = TRUE;
```

## 안티패턴
- 분석 쿼리마다 복잡한 조인을 다시 작성 — mart 테이블 부재
- 의미 없는 컬럼명: col1, val, flag — 도메인 언어로 명명
- NULL과 빈 문자열 혼용 — 일관성 없음
- 파티션 없는 대용량 테이블 — 전체 스캔 필연
- 변경 이력 없이 현재 값만 저장 — 과거 분석 불가

## 실전 팁
- 명명 규칙: fact_*, dim_*, stg_*, mart_* 접두사
- 날짜 차원 테이블(dim_date): 한국 공휴일, 영업일 포함
- SCD Type 2: 세그먼트 변경 이력이 있어야 코호트 분석 가능
- 증분 업데이트 컬럼: updated_at 필수 (증분 적재 기준)
- 데이터 카탈로그: 테이블/컬럼 설명을 dbt 문서로 자동화
