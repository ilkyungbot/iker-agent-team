# 지표(Metrics) — 올바른 지표를 고르는 기준

> 모든 지표는 의사결정을 위해 존재한다. 측정 가능하다고 의미 있는 것이 아니다.

## 핵심 원칙
- North Star Metric: 제품의 핵심 가치를 하나의 숫자로 표현
- Input Metric(선행) vs Output Metric(후행)을 구분하라
- Vanity Metric(허영 지표)을 경계하라 — 앱 다운로드 수, 총 가입자 수
- 지표는 분해 가능해야 한다 (드릴다운 구조)
- 지표 정의(분자/분모/기간/대상)를 문서화하라

## 판단 기준
| 지표 유형 | 예시 | 용도 |
|-----------|------|------|
| 성장 지표 | DAU, MAU, 신규 가입 | 규모 추적 |
| 참여 지표 | 세션 길이, 기능 사용률 | 제품 가치 검증 |
| 수익 지표 | ARPU, LTV, MRR | 비즈니스 건강도 |
| 품질 지표 | 에러율, 로딩 시간 | 서비스 안정성 |
| 효율 지표 | CAC, ROI, CPA | 투자 효율성 |

## 실전 예시

### 핵심 지표 SQL 계산
```sql
-- DAU / MAU (Stickiness)
WITH daily_active AS (
  SELECT event_date, COUNT(DISTINCT user_id) AS dau
  FROM events
  WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY event_date
),
monthly_active AS (
  SELECT COUNT(DISTINCT user_id) AS mau
  FROM events
  WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
)
SELECT
  AVG(d.dau) AS avg_dau,
  m.mau,
  ROUND(AVG(d.dau) / m.mau * 100, 1) AS stickiness_pct
FROM daily_active d, monthly_active m
GROUP BY m.mau;
```

```sql
-- ARPU (Average Revenue Per User)
SELECT
  DATE_TRUNC(order_date, MONTH) AS month,
  SUM(amount) AS revenue,
  COUNT(DISTINCT user_id) AS paying_users,
  COUNT(DISTINCT all_users.user_id) AS total_active_users,
  ROUND(SUM(amount) / COUNT(DISTINCT user_id), 0) AS arppu,
  ROUND(SUM(amount) / COUNT(DISTINCT all_users.user_id), 0) AS arpu
FROM orders
JOIN (
  SELECT DISTINCT user_id,
    DATE_TRUNC(event_date, MONTH) AS month
  FROM events
) all_users USING (user_id)
  -- ... (실제로는 month 조건 추가)
GROUP BY 1
ORDER BY 1;
```

### Python으로 지표 트래킹 대시보드
```python
import pandas as pd

def calculate_growth_metrics(df, date_col, user_col):
    """월별 성장 지표 계산"""
    df[date_col] = pd.to_datetime(df[date_col])
    df['month'] = df[date_col].dt.to_period('M')

    monthly = df.groupby('month')[user_col].nunique().reset_index()
    monthly.columns = ['month', 'mau']
    monthly['mom_growth'] = monthly['mau'].pct_change() * 100

    return monthly

# LTV 계산
def calculate_ltv(orders_df, cohort_months=12):
    orders_df['order_month'] = pd.to_datetime(orders_df['order_date']).dt.to_period('M')
    orders_df['first_month'] = orders_df.groupby('user_id')['order_month'].transform('min')
    orders_df['month_since_first'] = (
        orders_df['order_month'] - orders_df['first_month']
    ).apply(lambda x: x.n)

    ltv = orders_df[orders_df['month_since_first'] < cohort_months]\
        .groupby('user_id')['amount'].sum()
    return ltv.median(), ltv.mean()
```

## 안티패턴
- 총 누적 수치만 보고하기 (항상 활성/신규 구분)
- 지표 정의 없이 "활성 사용자" 말하기
- 상관관계를 인과관계로 오해 (매출↑ = 기능 성공이 아님)
- 지표가 너무 많아서 아무것도 안 보이는 대시보드
- 목표치 없이 지표만 추적 (OKR/목표와 연결 필수)

## 실전 팁
- Metric Tree: NSM → 1차 지표 3-5개 → 2차 지표로 분해
- 지표 변동 시 먼저 데이터 품질 이슈 확인 후 인사이트 도출
- WoW(주간), MoM(월간), YoY(연간) — 계절성에 따라 비교 기간 선택
- 지표 소수점은 2자리 이하로 — 정밀도의 환상을 피하라
- "좋은 지표"는 행동을 바꿀 수 있는 지표
