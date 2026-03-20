# 세분화(Segmentation) — 사용자를 의미 있는 그룹으로 나누기

> 세분화는 평균의 함정을 피하는 방법이다. 올바른 세분화가 올바른 행동을 만든다.

## 핵심 원칙
- 세분화 목적(마케팅/제품/분석)에 따라 기준이 달라진다
- 세그먼트는 서로 배타적이고 전체를 포함해야 한다 (MECE)
- 세그먼트 크기가 너무 작으면 노이즈, 너무 크면 의미 없음
- 행동 기반 세분화가 인구통계 기반보다 더 예측력이 높다
- 세그먼트는 시간에 따라 변한다 — 동적 세분화를 고려하라

## 판단 기준
| 세분화 유형 | 기준 | 활용 |
|-------------|------|------|
| 인구통계 | 나이, 성별, 지역 | 광고 타겟팅 |
| 행동 | 사용 패턴, 구매 이력 | 개인화, 리텐션 |
| 가치 기반 | LTV, 수익 기여 | 우선순위 리소스 배분 |
| 니즈 기반 | 사용 목적, Pain Point | 제품 개발 방향 |
| RFM | Recency/Frequency/Monetary | CRM, 마케팅 |

## 실전 예시

### RFM 세분화
```sql
WITH rfm_base AS (
  SELECT
    user_id,
    DATE_DIFF(CURRENT_DATE(), MAX(order_date), DAY) AS recency,
    COUNT(DISTINCT order_id) AS frequency,
    SUM(amount) AS monetary
  FROM orders
  WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
  GROUP BY user_id
),
rfm_scores AS (
  SELECT
    user_id,
    recency, frequency, monetary,
    NTILE(5) OVER (ORDER BY recency ASC)   AS r_score,  -- 낮을수록 높은 점수
    NTILE(5) OVER (ORDER BY frequency DESC) AS f_score,
    NTILE(5) OVER (ORDER BY monetary DESC)  AS m_score
  FROM rfm_base
)
SELECT
  user_id,
  r_score, f_score, m_score,
  r_score + f_score + m_score AS rfm_total,
  CASE
    WHEN r_score >= 4 AND f_score >= 4 THEN 'Champions'
    WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal Customers'
    WHEN r_score >= 4 AND f_score <= 2 THEN 'Recent Customers'
    WHEN r_score <= 2 AND f_score >= 3 THEN 'At Risk'
    WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost Customers'
    ELSE 'Potential Loyalists'
  END AS rfm_segment
FROM rfm_scores;
```

### K-Means 클러스터링 (Python)
```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score
import matplotlib.pyplot as plt

def find_optimal_clusters(X_scaled, max_k=10):
    """엘보우 + 실루엣으로 최적 k 탐색"""
    inertias = []
    silhouette_scores = []

    for k in range(2, max_k + 1):
        kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
        labels = kmeans.fit_predict(X_scaled)
        inertias.append(kmeans.inertia_)
        silhouette_scores.append(silhouette_score(X_scaled, labels))

    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))
    ax1.plot(range(2, max_k + 1), inertias, 'bo-')
    ax1.set_title('엘보우 방법')
    ax1.set_xlabel('클러스터 수 (k)')
    ax1.set_ylabel('관성(Inertia)')

    ax2.plot(range(2, max_k + 1), silhouette_scores, 'ro-')
    ax2.set_title('실루엣 점수')
    ax2.set_xlabel('클러스터 수 (k)')

    plt.tight_layout()
    return inertias, silhouette_scores

def segment_users(features_df, feature_cols, n_clusters=4):
    X = features_df[feature_cols].fillna(0)
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)

    kmeans = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
    features_df['cluster'] = kmeans.fit_predict(X_scaled)

    # 클러스터 프로파일
    profile = features_df.groupby('cluster')[feature_cols].agg(['mean', 'median', 'count'])
    print("클러스터 프로파일:")
    print(profile)

    return features_df, kmeans, scaler
```

### 세그먼트 분석 리포트
```sql
-- 세그먼트별 핵심 지표 비교
SELECT
  rfm_segment,
  COUNT(*) AS user_count,
  ROUND(COUNT(*) / SUM(COUNT(*)) OVER () * 100, 1) AS pct_of_users,
  ROUND(AVG(monetary), 0) AS avg_revenue,
  ROUND(SUM(monetary), 0) AS total_revenue,
  ROUND(SUM(monetary) / SUM(SUM(monetary)) OVER () * 100, 1) AS pct_of_revenue,
  ROUND(AVG(frequency), 1) AS avg_orders,
  ROUND(AVG(recency), 0) AS avg_days_since_order
FROM rfm_segments
GROUP BY rfm_segment
ORDER BY total_revenue DESC;
```

## 안티패턴
- 세그먼트 기준을 나중에 바꾸기 — 비교 불가
- 세그먼트 수가 너무 많음 (20개+) — 실행 불가
- 클러스터링 후 세그먼트에 이름 없이 "클러스터 1, 2, 3"
- 세분화 결과를 마케팅에만 쓰고 제품에 반영 안 함
- 정적 세분화 — 사용자 행동이 바뀌어도 세그먼트 안 바뀜

## 실전 팁
- 세그먼트 이름은 행동을 묘사해야 한다: "고액 충성 고객", "재활성화 필요 고객"
- 동적 세그먼트: SQL scheduled query로 주간 자동 업데이트
- 세그먼트 크기 변화 추적: 세그먼트 이동이 건강도 지표
- 세그먼트별 LTV 예측: 어떤 세그먼트에 투자할지 결정
- 페르소나와 연결: 정량 세그먼트 + 정성 인터뷰 = 완전한 이해
