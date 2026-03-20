# Python 분석 — 데이터 분석가를 위한 Python 실전 패턴

> Python은 분석가의 망치다. 모든 것을 망치로 다루려 하지 말고, 적재적소에 써라.

## 핵심 원칙
- Pandas는 메모리 내 처리, 대용량은 BigQuery/Spark로
- 재현 가능한 분석: 노트북보다 스크립트, 랜덤 시드 고정
- 함수로 캡슐화: 같은 코드를 두 번 쓰면 함수로 만들어라
- 데이터 탐색 → 가설 → 검증 순서를 지켜라
- 타입 힌트와 독스트링으로 코드를 자문서화하라

## 판단 기준
| 라이브러리 | 용도 | 대안 |
|------------|------|------|
| pandas | 테이블 데이터 처리 | polars (대용량, 빠름) |
| matplotlib | 기본 시각화 | plotly (인터랙티브) |
| seaborn | 통계 시각화 | altair (선언형) |
| scikit-learn | ML 모델링 | xgboost, lightgbm |
| scipy | 통계 검정 | statsmodels |

## 실전 예시

### 분석 노트북 기본 구조
```python
# 1. 환경 설정
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from google.cloud import bigquery

pd.set_option('display.max_columns', 50)
pd.set_option('display.float_format', '{:,.2f}'.format)
plt.rcParams['figure.figsize'] = (12, 5)
plt.rcParams['font.family'] = 'AppleGothic'  # 한글 폰트

RANDOM_SEED = 42
np.random.seed(RANDOM_SEED)

# 2. 데이터 로드
client = bigquery.Client(project='your-project')

def load_data(query: str) -> pd.DataFrame:
    """BigQuery에서 데이터 로드 + 캐싱"""
    return client.query(query).to_dataframe()

# 3. EDA (탐색적 데이터 분석)
def quick_eda(df: pd.DataFrame) -> None:
    print(f"Shape: {df.shape}")
    print(f"\n데이터 타입:\n{df.dtypes}")
    print(f"\n결측값:\n{df.isnull().sum()[df.isnull().sum() > 0]}")
    print(f"\n기술통계:\n{df.describe()}")
```

### Pandas 핵심 패턴
```python
import pandas as pd
import numpy as np

# 체이닝으로 가독성 있는 데이터 처리
result = (
    df
    .query("country == 'KR' and amount > 0")
    .assign(
        order_month=lambda x: pd.to_datetime(x['order_date']).dt.to_period('M'),
        revenue_tier=lambda x: pd.cut(
            x['amount'],
            bins=[0, 10000, 50000, 100000, np.inf],
            labels=['low', 'mid', 'high', 'premium']
        )
    )
    .groupby(['order_month', 'revenue_tier'], observed=True)
    .agg(
        n_orders=('order_id', 'count'),
        total_revenue=('amount', 'sum'),
        unique_buyers=('user_id', 'nunique')
    )
    .reset_index()
)

# apply 대신 vectorized 연산 (10x 빠름)
# 나쁜 예:
df['is_high_value'] = df['amount'].apply(lambda x: x > 100000)
# 좋은 예:
df['is_high_value'] = df['amount'] > 100000

# groupby transform (집계 후 원본 크기 유지)
df['user_total_revenue'] = df.groupby('user_id')['amount'].transform('sum')
df['revenue_pct'] = df['amount'] / df['user_total_revenue']

# 효율적인 날짜 처리
df['order_date'] = pd.to_datetime(df['order_date'])
df['days_since_order'] = (pd.Timestamp.now() - df['order_date']).dt.days
```

### 재사용 가능한 분석 함수 패턴
```python
from typing import Optional, List
import pandas as pd

def cohort_retention(
    df: pd.DataFrame,
    user_col: str = 'user_id',
    date_col: str = 'event_date',
    cohort_freq: str = 'M'
) -> pd.DataFrame:
    """
    코호트 리텐션 계산

    Args:
        df: user_id, event_date 포함 이벤트 데이터프레임
        user_col: 사용자 ID 컬럼명
        date_col: 이벤트 날짜 컬럼명
        cohort_freq: 코호트 주기 ('M': 월, 'W': 주)

    Returns:
        코호트별 리텐션율 피벗 테이블
    """
    df = df.copy()
    df[date_col] = pd.to_datetime(df[date_col])
    df['cohort'] = df.groupby(user_col)[date_col]\
        .transform('min').dt.to_period(cohort_freq)
    df['period'] = df[date_col].dt.to_period(cohort_freq)
    df['period_number'] = (df['period'] - df['cohort']).apply(lambda x: x.n)

    cohort_data = df.groupby(['cohort', 'period_number'])[user_col]\
        .nunique().reset_index()
    cohort_sizes = cohort_data[cohort_data['period_number'] == 0]\
        .set_index('cohort')[user_col]

    cohort_data['retention_rate'] = cohort_data.apply(
        lambda r: r[user_col] / cohort_sizes[r['cohort']], axis=1
    )

    return cohort_data.pivot(
        index='cohort', columns='period_number', values='retention_rate'
    )
```

## 안티패턴
- Jupyter 노트북 전체가 하나의 셀 — 재실행 불가
- `df = df` 방식 수정 — `df.copy()` 또는 `inplace=True` 명확히
- apply 루프로 10만 행 처리 — 벡터화 연산 사용
- 하드코딩된 경로 — 환경변수 또는 설정 파일 사용
- 전처리 코드를 노트북에만 — 재사용 불가

## 실전 팁
- `%%time` 또는 `%timeit`으로 셀 실행시간 측정
- `pd.read_csv(file, dtype={...}, parse_dates=[...])`로 타입 처음부터 지정
- `df.sample(1000)` + 빠른 검증 후 전체 데이터 적용
- Polars: pandas 대비 5-10x 빠른 대용량 처리
- `tqdm`으로 긴 루프 진행 상황 표시
