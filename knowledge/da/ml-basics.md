# ML 기초 — 데이터 분석가를 위한 머신러닝 입문

> ML은 마법이 아니라 통계의 확장이다. 문제 정의가 모델 선택보다 중요하다.

## 핵심 원칙
- 문제 유형(분류/회귀/클러스터링)을 먼저 정의하라
- 단순한 모델이 복잡한 모델보다 먼저다 (로지스틱 회귀 → 트리 → 앙상블)
- 피처 엔지니어링이 모델 선택보다 더 중요하다
- 검증(Validation) 없는 모델은 의미 없다
- 해석 가능성(Explainability)이 비즈니스에서는 중요하다

## 판단 기준
| 문제 유형 | 예시 | 시작 모델 |
|-----------|------|-----------|
| 이진 분류 | 이탈 예측, 사기 탐지 | Logistic Regression |
| 다중 분류 | 카테고리 분류 | Random Forest |
| 회귀 | 매출 예측, 가격 예측 | Linear Regression → XGBoost |
| 클러스터링 | 고객 세분화 | K-Means → DBSCAN |
| 이상 탐지 | 데이터 품질, 사기 | Isolation Forest |

## 실전 예시

### 이탈 예측 모델 (엔드투엔드)
```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split, StratifiedKFold
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import classification_report, roc_auc_score
import shap

# 1. 피처 엔지니어링
def create_churn_features(events_df, orders_df, as_of_date):
    features = events_df[events_df['event_date'] <= as_of_date]\
        .groupby('user_id').agg(
            recency=('event_date', lambda x: (pd.Timestamp(as_of_date) -
                                               pd.to_datetime(x.max())).days),
            frequency=('event_date', 'nunique'),
            total_events=('event_id', 'count'),
            feature_breadth=('event_name', 'nunique')
        )

    revenue_features = orders_df[orders_df['order_date'] <= as_of_date]\
        .groupby('user_id').agg(
            total_revenue=('amount', 'sum'),
            order_count=('order_id', 'count'),
            avg_order_value=('amount', 'mean')
        )

    return features.join(revenue_features, how='left').fillna(0)

# 2. 모델 학습 및 평가
def train_churn_model(X, y):
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, stratify=y, random_state=42
    )

    scaler = StandardScaler()
    X_train_scaled = scaler.fit_transform(X_train)
    X_test_scaled = scaler.transform(X_test)

    models = {
        'Logistic Regression': LogisticRegression(max_iter=1000),
        'GBM': GradientBoostingClassifier(n_estimators=100)
    }

    for name, model in models.items():
        model.fit(X_train_scaled, y_train)
        y_pred = model.predict(X_test_scaled)
        y_prob = model.predict_proba(X_test_scaled)[:, 1]
        print(f"\n{name}")
        print(f"AUC-ROC: {roc_auc_score(y_test, y_prob):.3f}")
        print(classification_report(y_test, y_pred))

    return models

# 3. SHAP으로 모델 해석
def explain_model(model, X, feature_names):
    explainer = shap.TreeExplainer(model)
    shap_values = explainer.shap_values(X)

    shap.summary_plot(shap_values, X,
                      feature_names=feature_names,
                      plot_type='bar')
    return shap_values
```

### 교차 검증으로 모델 안정성 확인
```python
from sklearn.model_selection import cross_val_score

def evaluate_model_stability(model, X, y, cv=5):
    scores = cross_val_score(model, X, y,
                             cv=StratifiedKFold(n_splits=cv, shuffle=True),
                             scoring='roc_auc')
    print(f"AUC: {scores.mean():.3f} (+/- {scores.std() * 2:.3f})")
    return scores
```

## 안티패턴
- 전체 데이터로 학습하고 같은 데이터로 평가 (Data Leakage)
- Accuracy만 보기 — 불균형 데이터에서 쓸모없음 (AUC, F1 사용)
- 하이퍼파라미터 튜닝에 집착하고 피처 개선 무시
- 테스트 세트를 여러 번 사용하여 오버피팅된 평가
- 프로덕션 데이터 분포와 학습 데이터 분포 불일치 무시

## 실전 팁
- 불균형 데이터: class_weight='balanced' 또는 SMOTE 오버샘플링
- 피처 중요도 먼저 확인 → 불필요한 피처 제거 → 모델 단순화
- 비즈니스 임계값 조정: Precision-Recall 트레이드오프 비즈니스 관점으로 결정
- 모델 모니터링: 프로덕션 후 예측 분포 변화 추적 (Data Drift)
- MLflow로 실험 추적: 파라미터, 메트릭, 모델 아티팩트 버전 관리
