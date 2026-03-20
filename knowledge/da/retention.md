# 리텐션 분석 — 사용자가 돌아오는 이유를 이해하기

> 신규 획득보다 리텐션이 10배 저렴하다. 리텐션이 비즈니스의 실질적 건강도다.

## 핵심 원칙
- 리텐션 정의를 명확히 하라 (1일, 7일, 30일 리텐션은 각기 다른 의미)
- 리텐션 기준 행동을 의미 있는 것으로 정의하라 (로그인 vs 핵심 행동)
- 리텐션은 제품-시장 적합성(PMF)의 가장 정직한 지표
- 세그먼트별 리텐션 차이가 개선 방향을 알려준다
- 이탈 시점과 이탈 직전 행동 패턴이 핵심 인사이트

## 판단 기준
| 리텐션 유형 | 정의 | 벤치마크 (SaaS 기준) |
|-------------|------|---------------------|
| Day 1 | 가입 다음날 재방문 | 40%+ 양호 |
| Day 7 | 가입 7일 후 재방문 | 20%+ 양호 |
| Day 30 | 가입 30일 후 재방문 | 10%+ 양호 |
| L28 리텐션 | 지난 4주 중 3주 이상 방문 | 소셜앱 기준 |

## 실전 예시

### N일 리텐션 계산
```sql
WITH first_active AS (
  SELECT
    user_id,
    MIN(event_date) AS first_date
  FROM events
  GROUP BY user_id
),
subsequent_activity AS (
  SELECT DISTINCT
    e.user_id,
    DATE_DIFF(e.event_date, f.first_date, DAY) AS days_since_first
  FROM events e
  JOIN first_active f USING (user_id)
  WHERE e.event_date > f.first_date
)
SELECT
  days_since_first,
  COUNT(DISTINCT user_id) AS retained_users,
  COUNT(DISTINCT user_id) / (
    SELECT COUNT(DISTINCT user_id) FROM first_active
  ) AS retention_rate
FROM subsequent_activity
WHERE days_since_first IN (1, 3, 7, 14, 30, 60, 90)
GROUP BY 1
ORDER BY 1;
```

### 리텐션 곡선 분석 (Python)
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from scipy.optimize import curve_fit

def retention_curve(t, a, b):
    """지수 감쇠 모델 (a: 초기 리텐션, b: 감쇠율)"""
    return a * np.exp(-b * t)

def analyze_retention(retention_df):
    """
    retention_df: days_since_first, retention_rate 컬럼
    """
    days = retention_df['days_since_first'].values
    rates = retention_df['retention_rate'].values

    # 커브 피팅
    popt, _ = curve_fit(retention_curve, days, rates,
                        p0=[0.5, 0.05], maxfev=5000)

    # 예측 리텐션
    days_future = np.arange(0, 365)
    predicted = retention_curve(days_future, *popt)

    # 리텐션 플래토 (안정화 지점)
    plateau_idx = np.where(np.diff(predicted) > -0.001)[0]
    plateau_rate = predicted[plateau_idx[0]] if len(plateau_idx) > 0 else predicted[-1]

    print(f"초기 리텐션: {popt[0]:.1%}")
    print(f"감쇠율: {popt[1]:.4f}")
    print(f"안정화 리텐션: {plateau_rate:.1%}")

    return predicted

# 리텐션 개선 세그먼트 분석
def retention_by_segment(events_df, segment_col):
    """세그먼트별 7일 리텐션 비교"""
    events_df['event_date'] = pd.to_datetime(events_df['event_date'])

    first_dates = events_df.groupby('user_id').agg(
        first_date=('event_date', 'min'),
        segment=(segment_col, 'first')
    ).reset_index()

    day7_active = events_df[
        events_df['event_date'].isin(
            first_dates.apply(
                lambda r: r['first_date'] + pd.Timedelta(days=7), axis=1
            )
        )
    ]['user_id'].unique()

    first_dates['retained_d7'] = first_dates['user_id'].isin(day7_active)

    return first_dates.groupby('segment')['retained_d7'].agg(['mean', 'count'])
```

### 이탈 예측 특성 엔지니어링
```sql
-- 이탈 가능성 높은 사용자 특성
SELECT
  user_id,
  DATEDIFF(CURRENT_DATE, MAX(event_date)) AS days_since_last_active,
  COUNT(DISTINCT event_date) AS active_days_last_30,
  COUNT(*) AS total_events_last_30,
  COUNT(DISTINCT event_name) AS feature_breadth
FROM events
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY user_id
HAVING days_since_last_active > 14  -- 14일 미방문 = 이탈 위험군
ORDER BY days_since_last_active DESC;
```

## 안티패턴
- 리텐션 정의 없이 "리텐션이 올랐다/내렸다" 보고
- 전체 사용자 리텐션만 보고 신규 유입 급증으로 인한 희석 무시
- 리텐션 분석을 코호트 없이 스냅샷으로만 계산
- 리텐션 하락 시 즉시 알림 기능 구현 — 원인 분석 먼저
- 리텐션 지표 하나만 보고 (Day7 올랐지만 Day30은 내렸을 수도)

## 실전 팁
- Aha Moment: "이것을 경험한 사람이 리텐션이 높다"를 찾아라
- 리텐션 개선 실험: 온보딩 플로우, 알림, 넛지가 효과적
- L28 리텐션 (Lyft 방식): 지난 28일 중 최소 X일 활동 = 더 안정적 지표
- 이탈 사용자 인터뷰: 데이터가 못 보여주는 "왜"를 찾는 유일한 방법
- 리텐션 vs 인게이지먼트: 돌아오는가(리텐션) + 충분히 사용하는가(인게이지먼트)
