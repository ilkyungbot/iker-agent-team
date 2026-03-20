# 통계 기초 — 데이터 분석가가 반드시 알아야 할 핵심

> 통계는 불확실성을 정량화하는 언어다. 숫자 하나에 맥락이 담겨야 한다.

## 핵심 원칙
- 평균은 분포를 숨긴다. 항상 분산/중위수/분포 형태를 함께 확인
- 상관관계 ≠ 인과관계. 제3변수(교란변수)를 항상 의심하라
- 표본 크기가 크면 통계적으로 유의하지만 실용적으로 무의미할 수 있다
- p-value는 "효과가 있다"는 증거가 아니라 "우연이 아닐 가능성"이다
- 신뢰구간은 p-value보다 더 많은 정보를 담고 있다

## 판단 기준
| 상황 | 적합한 통계 기법 |
|------|-----------------|
| 두 그룹 평균 비교 | t-test (정규분포 가정) |
| 두 그룹 전환율 비교 | 카이제곱 / z-test for proportions |
| 비정규 분포 비교 | Mann-Whitney U test |
| 세 그룹 이상 비교 | ANOVA → Post-hoc (Tukey) |
| 변수 간 관계 | 피어슨 / 스피어만 상관계수 |

## 실전 예시

### 기술통계 요약
```python
import pandas as pd
import numpy as np
from scipy import stats

def describe_distribution(series, name=""):
    print(f"=== {name} 분포 요약 ===")
    print(f"N: {len(series):,}")
    print(f"평균: {series.mean():,.2f}")
    print(f"중위수: {series.median():,.2f}")
    print(f"표준편차: {series.std():,.2f}")
    print(f"왜도(Skewness): {series.skew():.3f}")
    print(f"첨도(Kurtosis): {series.kurtosis():.3f}")
    print(f"5th ~ 95th 백분위: {series.quantile(0.05):,.0f} ~ {series.quantile(0.95):,.0f}")

    # 정규성 검정 (n < 5000)
    if len(series) < 5000:
        stat, p = stats.shapiro(series.dropna())
        print(f"Shapiro-Wilk p-value: {p:.4f} ({'정규분포 아님' if p < 0.05 else '정규분포 가능성'})")
```

### 신뢰구간 계산
```python
def confidence_interval(data, confidence=0.95):
    n = len(data)
    mean = np.mean(data)
    se = stats.sem(data)
    interval = se * stats.t.ppf((1 + confidence) / 2., n - 1)
    return mean - interval, mean + interval

# 비율의 신뢰구간 (Wilson Score)
def proportion_confidence_interval(successes, total, confidence=0.95):
    from statsmodels.stats.proportion import proportion_confint
    lower, upper = proportion_confint(successes, total, alpha=1-confidence, method='wilson')
    return lower, upper

lower, upper = proportion_confidence_interval(480, 10000)
print(f"전환율 95% CI: {lower:.2%} ~ {upper:.2%}")
```

### 이상치 탐지
```python
def detect_outliers(series, method='iqr'):
    if method == 'iqr':
        Q1, Q3 = series.quantile([0.25, 0.75])
        IQR = Q3 - Q1
        lower = Q1 - 1.5 * IQR
        upper = Q3 + 1.5 * IQR
    elif method == 'zscore':
        z_scores = np.abs(stats.zscore(series.dropna()))
        return series[z_scores > 3]

    outliers = series[(series < lower) | (series > upper)]
    print(f"이상치 비율: {len(outliers)/len(series):.1%} ({len(outliers)}건)")
    return outliers
```

## 안티패턴
- 평균만 보고하기 — 수익, 체류시간 등은 심하게 치우친 분포
- p < 0.05이면 무조건 "효과 있음" 결론 — 효과 크기(Cohen's d) 확인 필수
- 다중 비교 무시 — 20개 지표 보면 1개는 우연히 유의
- 소표본(n<30)에서 정규 분포 가정 — 비모수 검정 고려
- 데이터 정제 전 통계 계산 — 이상치/결측값이 결과 왜곡

## 실전 팁
- 수익/매출 데이터는 로그 변환 후 분석 (로그정규분포)
- Bootstrap으로 소표본에서 신뢰구간 추정 가능
- Effect Size: Cohen's d (평균 차이), Cramér's V (범주형)
- 중앙값 + IQR이 평균 + 표준편차보다 강건한 요약 통계
- 시각화: 박스플롯 > 막대그래프 (분포 형태가 보임)
