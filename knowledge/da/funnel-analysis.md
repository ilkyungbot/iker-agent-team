# 퍼널 분석 — 전환 병목을 찾는 체계적 방법

> 퍼널은 숫자가 아니라 이야기다. 어디서 누가 왜 이탈하는지를 찾아라.

## 핵심 원칙
- 퍼널은 순서가 있는 이벤트 시퀀스다. 순서 조건을 명확히 정의하라
- 전환율만 보지 말고, 이탈자의 특성을 분석하라
- 시간 창(time window)을 반드시 설정하라 (예: 7일 내 전환)
- 코호트별로 비교해야 의미 있다 (신규 vs 재방문, 채널별)
- 퍼널 단계 간 소요시간도 핵심 지표다

## 판단 기준
| 퍼널 유형 | 특징 | 적합한 분석 |
|-----------|------|-------------|
| 선형 퍼널 | 단계 순서 고정 | 단계별 전환율, 이탈률 |
| 비선형 퍼널 | 단계 순서 유연 | 경로 분석, 최단 경로 |
| 복귀 퍼널 | 이탈 후 복귀 | 재진입률, 복귀 트리거 |

## 실전 예시

### 기본 퍼널 (단계별 전환율)
```sql
WITH funnel_steps AS (
  SELECT
    user_id,
    MAX(CASE WHEN event_name = 'page_view' THEN 1 ELSE 0 END) AS step1,
    MAX(CASE WHEN event_name = 'add_to_cart' THEN 1 ELSE 0 END) AS step2,
    MAX(CASE WHEN event_name = 'checkout_start' THEN 1 ELSE 0 END) AS step3,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS step4
  FROM events
  WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'
  GROUP BY user_id
)
SELECT
  SUM(step1) AS view_users,
  SUM(step2) AS cart_users,
  SUM(step3) AS checkout_users,
  SUM(step4) AS purchase_users,
  ROUND(SUM(step2) / SUM(step1) * 100, 1) AS view_to_cart_pct,
  ROUND(SUM(step3) / SUM(step2) * 100, 1) AS cart_to_checkout_pct,
  ROUND(SUM(step4) / SUM(step3) * 100, 1) AS checkout_to_purchase_pct
FROM funnel_steps;
```

### 순서 보장 퍼널 (시간 창 포함)
```sql
WITH ordered_events AS (
  SELECT
    user_id,
    event_name,
    event_timestamp,
    MIN(CASE WHEN event_name = 'signup' THEN event_timestamp END)
      OVER (PARTITION BY user_id) AS signup_time
  FROM events
)
SELECT
  COUNT(DISTINCT CASE WHEN event_name = 'signup' THEN user_id END) AS signups,
  COUNT(DISTINCT CASE
    WHEN event_name = 'first_purchase'
     AND event_timestamp <= TIMESTAMP_ADD(signup_time, INTERVAL 7 DAY)
    THEN user_id
  END) AS converted_in_7d
FROM ordered_events;
```

### Python으로 퍼널 시각화
```python
import pandas as pd
import matplotlib.pyplot as plt

funnel_data = {
    'stage': ['방문', '회원가입', '장바구니', '결제완료'],
    'users': [10000, 3200, 1800, 720]
}
df = pd.DataFrame(funnel_data)
df['conversion_rate'] = df['users'] / df['users'].iloc[0] * 100
df['step_rate'] = df['users'] / df['users'].shift(1) * 100

fig, ax = plt.subplots(figsize=(10, 6))
bars = ax.barh(df['stage'][::-1], df['users'][::-1], color='steelblue')
for i, (users, rate) in enumerate(zip(df['users'][::-1], df['conversion_rate'][::-1])):
    ax.text(users + 100, i, f'{users:,}명 ({rate:.1f}%)', va='center')
plt.title('구매 전환 퍼널')
plt.tight_layout()
plt.show()
```

## 안티패턴
- 시간 창 없이 전체 기간 집계 — 신규/기존 유저 섞임
- 단계 순서 무시하고 단순 이벤트 유무만 체크
- 전체 평균만 보고 세그먼트 분석 생략
- 이탈률만 보고 이탈 사용자 특성 분석 안 함
- A/B 테스트 없이 퍼널 개선 효과 주장

## 실전 팁
- 퍼널 단계 간 중위 소요시간(median time)을 함께 리포트
- 이탈 직전 행동 패턴 분석 → 이탈 예측 모델 입력 특성
- 디바이스/채널/코호트별 퍼널 비교가 핵심 인사이트
- Mixpanel, Amplitude의 퍼널 기능은 시각화에 활용하되, 커스텀 분석은 SQL로
- 퍼널 개선 실험 시 반드시 통계적 유의성 검증
