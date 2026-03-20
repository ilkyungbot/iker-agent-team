# 기여도 분석(Attribution) — 마케팅 채널의 진짜 기여 측정

> 마케팅 기여도는 진실이 아니라 모델이다. 모델의 한계를 알고 사용하라.

## 핵심 원칙
- 어떤 기여도 모델도 완벽하지 않다. 비즈니스 맥락에 맞는 모델을 선택하라
- 터치포인트 수집이 정확해야 기여도 분석이 의미 있다
- First-touch와 Last-touch는 극단적 단순화다 — 멀티터치 고려
- 쿠키리스 환경에서 어트리뷰션은 점점 어려워지고 있다
- 인크리멘탈리티(순증분 효과) 측정이 궁극적 목표

## 판단 기준
| 모델 | 특징 | 적합한 상황 |
|------|------|-------------|
| Last Touch | 마지막 채널 100% 기여 | 직접 반응 캠페인 |
| First Touch | 첫 채널 100% 기여 | 브랜드 인지도 측정 |
| Linear | 모든 채널 균등 배분 | 채널 역할 불분명할 때 |
| Time Decay | 전환에 가까울수록 높은 가중치 | 긴 구매 여정 |
| Data-Driven | ML 기반 기여 계산 | 충분한 전환 데이터 보유 시 |

## 실전 예시

### 멀티터치 기여도 SQL
```sql
-- 사용자별 전환 여정 추출
WITH user_journeys AS (
  SELECT
    user_id,
    conversion_id,
    conversion_date,
    conversion_value,
    channel,
    touchpoint_date,
    ROW_NUMBER() OVER (
      PARTITION BY user_id, conversion_id
      ORDER BY touchpoint_date
    ) AS touch_order,
    COUNT(*) OVER (
      PARTITION BY user_id, conversion_id
    ) AS total_touches
  FROM attribution_touchpoints
  WHERE touchpoint_date <= conversion_date
    AND touchpoint_date >= DATE_SUB(conversion_date, INTERVAL 30 DAY)
),
-- Linear Attribution
linear_attribution AS (
  SELECT
    channel,
    SUM(conversion_value / total_touches) AS attributed_revenue,
    COUNT(DISTINCT conversion_id) AS attributed_conversions
  FROM user_journeys
  GROUP BY channel
),
-- Time Decay Attribution
time_decay AS (
  SELECT
    channel,
    -- 최근 터치포인트에 가중치 (지수 감쇠)
    SUM(
      conversion_value *
      POW(0.7, total_touches - touch_order) /
      SUM(POW(0.7, total_touches - touch_order))
        OVER (PARTITION BY user_id, conversion_id)
    ) AS attributed_revenue
  FROM user_journeys
  GROUP BY channel
)
SELECT
  channel,
  la.attributed_revenue AS linear_revenue,
  td.attributed_revenue AS time_decay_revenue
FROM linear_attribution la
JOIN time_decay td USING (channel)
ORDER BY la.attributed_revenue DESC;
```

### Python 기여도 분석
```python
import pandas as pd
import numpy as np

def calculate_attribution(journeys_df, model='linear'):
    """
    journeys_df: user_id, conversion_id, channel,
                 touchpoint_order, total_touches, conversion_value
    """
    df = journeys_df.copy()

    if model == 'linear':
        df['weight'] = 1 / df['total_touches']

    elif model == 'first_touch':
        df['weight'] = (df['touchpoint_order'] == 1).astype(float)

    elif model == 'last_touch':
        df['weight'] = (df['touchpoint_order'] == df['total_touches']).astype(float)

    elif model == 'time_decay':
        # 반감기: 7일
        df['weight'] = np.power(0.7, df['total_touches'] - df['touchpoint_order'])
        df['weight'] = df['weight'] / df.groupby('conversion_id')['weight'].transform('sum')

    elif model == 'position_based':
        # 40% 첫 터치, 40% 마지막 터치, 20% 나머지 균등
        conditions = [
            df['touchpoint_order'] == 1,
            df['touchpoint_order'] == df['total_touches'],
            df['total_touches'] == 1  # 단일 터치
        ]
        middle_weight = (0.2 / (df['total_touches'] - 2).clip(1))
        df['weight'] = np.where(conditions[2], 1.0,
                       np.where(conditions[0], 0.4,
                       np.where(conditions[1], 0.4, middle_weight)))
        df['weight'] = df['weight'] / df.groupby('conversion_id')['weight'].transform('sum')

    df['attributed_value'] = df['weight'] * df['conversion_value']
    return df.groupby('channel')['attributed_value'].sum().sort_values(ascending=False)

# 모델 비교
models = ['first_touch', 'last_touch', 'linear', 'time_decay', 'position_based']
comparison = pd.DataFrame({
    model: calculate_attribution(journeys_df, model)
    for model in models
})
print(comparison)
```

## 안티패턴
- Last-touch만 보고 퍼포먼스 채널에 과잉 투자 — 상단 퍼널 기여 무시
- 오가닉/다이렉트 채널을 집계에서 제외
- View-through 기여도 과대 설정 — 인상은 클릭과 다름
- 어트리뷰션 윈도우를 너무 길게 설정 — 관련 없는 이전 채널 포함
- 채널 간 시너지 무시 (멀티채널 노출 효과)

## 실전 팁
- MMM (Marketing Mix Modeling): 집계 레벨에서 채널 기여 측정
- Incrementality Test: 채널을 끄고 전환 변화 측정 (진짜 기여)
- GA4 Data-Driven Attribution: 전환 200개+ 이상 있을 때 신뢰
- 어트리뷰션 리포트는 "채널 최적화"용, 예산 배분의 유일한 근거 아님
- 오가닉 기여 측정: 브랜드 검색량 변화로 간접 측정
