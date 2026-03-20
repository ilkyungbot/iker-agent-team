# 코호트 분석 — 시간 기반 사용자 행동 패턴 추적

> 코호트는 평균의 거짓말을 걷어낸다. 같은 시기에 시작한 사람들의 행동을 따라가라.

## 핵심 원칙
- 코호트는 공통된 특성/시작 시점으로 묶인 사용자 그룹
- 가장 흔한 코호트: 가입 월 기준 코호트
- 코호트 분석의 핵심: 시간이 지남에 따라 행동이 어떻게 변하는가
- 코호트 간 비교로 제품 개선 효과를 측정할 수 있다
- 코호트 크기가 너무 작으면 노이즈가 크다 (최소 100명 이상 권장)

## 판단 기준
| 코호트 유형 | 정의 기준 | 활용 |
|-------------|-----------|------|
| 가입 코호트 | 가입 월 | 리텐션, LTV 추적 |
| 구매 코호트 | 첫 구매 월 | 재구매율, CLTV |
| 기능 코호트 | 특정 기능 사용 시점 | 기능 가치 측정 |
| 획득 코호트 | 유입 채널 | 채널별 품질 비교 |

## 실전 예시

### 기본 코호트 리텐션 쿼리
```sql
WITH user_cohorts AS (
  -- 각 사용자의 첫 구매 월 (코호트 기준)
  SELECT
    user_id,
    DATE_TRUNC(MIN(order_date), MONTH) AS cohort_month
  FROM orders
  GROUP BY user_id
),
user_activities AS (
  -- 각 사용자의 활동 월
  SELECT DISTINCT
    user_id,
    DATE_TRUNC(order_date, MONTH) AS activity_month
  FROM orders
),
cohort_data AS (
  SELECT
    c.cohort_month,
    DATE_DIFF(a.activity_month, c.cohort_month, MONTH) AS months_since_cohort,
    COUNT(DISTINCT a.user_id) AS active_users
  FROM user_cohorts c
  JOIN user_activities a USING (user_id)
  GROUP BY 1, 2
),
cohort_sizes AS (
  SELECT cohort_month, COUNT(*) AS cohort_size
  FROM user_cohorts
  GROUP BY cohort_month
)
SELECT
  cd.cohort_month,
  cs.cohort_size,
  cd.months_since_cohort,
  cd.active_users,
  ROUND(cd.active_users / cs.cohort_size * 100, 1) AS retention_rate
FROM cohort_data cd
JOIN cohort_sizes cs USING (cohort_month)
WHERE cd.months_since_cohort <= 12
ORDER BY cd.cohort_month, cd.months_since_cohort;
```

### Python으로 코호트 히트맵 시각화
```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

def plot_cohort_heatmap(df):
    """
    df: cohort_month, months_since_cohort, retention_rate 컬럼 필요
    """
    pivot = df.pivot_table(
        index='cohort_month',
        columns='months_since_cohort',
        values='retention_rate'
    )

    fig, ax = plt.subplots(figsize=(14, 8))
    sns.heatmap(
        pivot,
        annot=True, fmt='.0f',
        cmap='YlOrRd_r',
        vmin=0, vmax=100,
        linewidths=0.5,
        ax=ax
    )
    ax.set_title('코호트별 월별 리텐션율 (%)', fontsize=14, pad=15)
    ax.set_xlabel('가입 후 경과 월')
    ax.set_ylabel('가입 코호트')
    plt.tight_layout()
    return fig

# 코호트 LTV 계산
def calculate_cohort_ltv(orders_df):
    orders_df['cohort_month'] = orders_df.groupby('user_id')['order_date']\
        .transform('min').dt.to_period('M')
    orders_df['order_month'] = pd.to_datetime(orders_df['order_date']).dt.to_period('M')
    orders_df['months_since'] = (
        orders_df['order_month'] - orders_df['cohort_month']
    ).apply(lambda x: x.n)

    ltv_cohort = orders_df.groupby(['cohort_month', 'months_since'])['amount']\
        .sum().groupby(level=0).cumsum().reset_index()
    ltv_cohort.columns = ['cohort_month', 'months_since', 'cumulative_ltv']

    return ltv_cohort
```

## 안티패턴
- 코호트 크기 무시하고 작은 코호트 결론 도출
- 코호트 비교 시 구성 차이 무시 (신규 vs 기존 제품, 다른 마케팅 환경)
- Month 0 리텐션을 100%가 아닌 다른 값으로 정규화
- 이탈 코호트와 잔존 코호트를 섞어서 분석
- 코호트 분석 결과만 보고 개선 원인 귀인 없이 "좋아졌다" 결론

## 실전 팁
- 코호트 히트맵에서 대각선 패턴: 특정 달의 전체적인 이상 징후
- 코호트 크기를 히트맵 첫 열에 표시하라 (n= 참고용)
- 롤링 리텐션 vs 범위 리텐션 구분 (7일 내 재방문 vs 정확히 7일째 방문)
- 코호트 분석은 월별보다 주별이 초기 패턴 파악에 유리
- "어떤 코호트가 가장 좋은가?" → 해당 코호트 획득 채널 역추적
