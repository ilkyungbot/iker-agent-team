# 예측(Forecasting) — 시계열 데이터로 미래 추정하기

> 예측은 미래를 맞추는 것이 아니라 불확실성을 정량화하는 것이다.

## 핵심 원칙
- 모든 예측은 틀린다. 신뢰구간과 불확실성을 함께 제시하라
- 데이터를 먼저 탐색하라 (트렌드, 계절성, 이상치 파악)
- 단순 모델로 시작하라 (이동평균 → ARIMA → Prophet → ML)
- 비즈니스 캘린더(공휴일, 프로모션) 반드시 반영하라
- 예측 정확도는 MAPE, MAE, RMSE로 평가하되 비즈니스 맥락과 함께

## 판단 기준
| 상황 | 적합한 방법 |
|------|-------------|
| 단기 예측 (주/월) | 이동평균, 지수평활 |
| 계절성 강한 데이터 | Prophet, SARIMA |
| 다변량 예측 | XGBoost, LightGBM with lag features |
| 비선형 패턴 | Neural Prophet, LSTM |
| 빠른 베이스라인 | Prophet (Auto) |

## 실전 예시

### Prophet으로 매출 예측
```python
from prophet import Prophet
import pandas as pd
import matplotlib.pyplot as plt

def forecast_revenue(df, periods=90, freq='D'):
    """
    df: 'ds' (날짜), 'y' (값) 컬럼 필요
    """
    # 이상치 처리
    q_low, q_high = df['y'].quantile([0.01, 0.99])
    df['y'] = df['y'].clip(q_low, q_high)

    model = Prophet(
        yearly_seasonality=True,
        weekly_seasonality=True,
        daily_seasonality=False,
        seasonality_mode='multiplicative',  # 계절성이 비율로 변하는 경우
        changepoint_prior_scale=0.05,  # 트렌드 변화 민감도
    )

    # 한국 공휴일 추가
    from prophet.make_holidays import make_holidays_df
    kr_holidays = make_holidays_df(year_list=[2024, 2025], country='KR')
    model.add_country_holidays(country_name='KR')

    model.fit(df)

    future = model.make_future_dataframe(periods=periods, freq=freq)
    forecast = model.predict(future)

    # 정확도 평가 (학습 기간 내)
    train_pred = forecast[forecast['ds'].isin(df['ds'])]
    mape = ((df['y'].values - train_pred['yhat'].values) /
            df['y'].values).abs().mean() * 100
    print(f"학습 기간 MAPE: {mape:.1f}%")

    return forecast, model

# 시각화
def plot_forecast(model, forecast):
    fig = model.plot(forecast, figsize=(14, 6))
    plt.title('매출 예측 (90일)', fontsize=14)
    plt.xlabel('날짜')
    plt.ylabel('매출')
    plt.tight_layout()

    # 컴포넌트 분해
    fig2 = model.plot_components(forecast)
    return fig, fig2
```

### XGBoost 기반 시계열 예측 (Lag Features)
```python
import xgboost as xgb
from sklearn.metrics import mean_absolute_percentage_error

def create_lag_features(df, target_col, lag_days=[1, 7, 14, 28]):
    """시계열 → 지도학습 피처로 변환"""
    df = df.copy().sort_values('date')

    for lag in lag_days:
        df[f'lag_{lag}'] = df[target_col].shift(lag)

    # 롤링 통계
    df['rolling_mean_7d'] = df[target_col].shift(1).rolling(7).mean()
    df['rolling_std_7d'] = df[target_col].shift(1).rolling(7).std()
    df['rolling_mean_28d'] = df[target_col].shift(1).rolling(28).mean()

    # 날짜 피처
    df['day_of_week'] = df['date'].dt.dayofweek
    df['day_of_month'] = df['date'].dt.day
    df['month'] = df['date'].dt.month
    df['is_weekend'] = df['day_of_week'].isin([5, 6]).astype(int)

    return df.dropna()

# 예측 성능 평가
def evaluate_forecast(y_true, y_pred):
    mape = mean_absolute_percentage_error(y_true, y_pred) * 100
    mae = abs(y_true - y_pred).mean()
    rmse = ((y_true - y_pred) ** 2).mean() ** 0.5

    print(f"MAPE: {mape:.1f}%")
    print(f"MAE: {mae:,.0f}")
    print(f"RMSE: {rmse:,.0f}")
```

## 안티패턴
- 과거 평균으로 미래 예측 — 트렌드/계절성 무시
- 단일 포인트 예측만 제시 — 불확실성 숨김
- 테스트 기간 데이터를 학습에 사용 (Data Leakage)
- 이상치 처리 없이 모델 학습 — 예측 왜곡
- 외부 요인(경기, 마케팅 계획) 무시

## 실전 팁
- 예측 정확도 기준: MAPE 10% 이하 = 훌륭, 20% 이하 = 양호
- Walk-Forward Validation: 시계열은 교차검증이 아니라 순차 검증
- 앙상블: Prophet + XGBoost 평균이 단일 모델보다 안정적
- 예측 vs 계획 비교: 예측은 현재 트렌드, 계획은 의도된 목표
- BigQuery ML: `CREATE MODEL ... OPTIONS(model_type='ARIMA_PLUS')`로 빠른 시작
