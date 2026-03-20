# 데이터 시각화 — 올바른 차트를 선택하는 법

> 차트는 데이터를 예쁘게 만드는 것이 아니라 패턴을 드러내는 것이다.

## 핵심 원칙
- 차트 유형은 "무엇을 비교하는가"에 따라 결정된다
- 잉크 대 데이터 비율(Data-ink Ratio)을 높여라 — 불필요한 장식 제거
- 색상은 의미를 전달할 때만 사용하라
- 축은 항상 0에서 시작하거나 절단을 명시하라
- 제목은 "무엇이 보이는가"가 아니라 "무엇을 말하는가"로 작성

## 판단 기준
| 목적 | 추천 차트 | 피할 차트 |
|------|-----------|-----------|
| 시간에 따른 변화 | 선 그래프 | 막대 (연속적 시간) |
| 카테고리 비교 | 가로 막대 | 파이차트 |
| 비율/구성 | 누적 막대, 영역 차트 | 3D 파이 |
| 분포 확인 | 히스토그램, 박스플롯 | 막대 (평균만) |
| 두 변수 관계 | 산점도 | 선 그래프 |
| 지리 분포 | 지도 차트 | 테이블 |

## 실전 예시

### Python matplotlib/seaborn 실전 패턴
```python
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd

# 스타일 설정
plt.style.use('seaborn-v0_8-whitegrid')
COLORS = {'primary': '#2563EB', 'success': '#16A34A',
          'danger': '#DC2626', 'neutral': '#6B7280'}

# 1. 트렌드 + 목표선
def plot_metric_trend(df, metric_col, target=None):
    fig, ax = plt.subplots(figsize=(12, 5))
    ax.plot(df['date'], df[metric_col], color=COLORS['primary'],
            linewidth=2, label='실제')
    if target:
        ax.axhline(y=target, color=COLORS['danger'],
                   linestyle='--', label=f'목표 {target:,}')
    ax.fill_between(df['date'], df[metric_col], alpha=0.1,
                    color=COLORS['primary'])
    ax.set_title(f'{metric_col} 트렌드', fontsize=14, fontweight='bold')
    ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'{x:,.0f}'))
    plt.legend()
    plt.tight_layout()
    return fig

# 2. 세그먼트 비교 (가로 막대)
def plot_segment_comparison(df, category_col, value_col, top_n=10):
    df_sorted = df.nlargest(top_n, value_col)
    fig, ax = plt.subplots(figsize=(10, 6))
    colors = [COLORS['primary'] if i == 0 else COLORS['neutral']
              for i in range(len(df_sorted))]
    bars = ax.barh(df_sorted[category_col], df_sorted[value_col], color=colors)
    for bar, val in zip(bars, df_sorted[value_col]):
        ax.text(bar.get_width() + max(df_sorted[value_col]) * 0.01,
                bar.get_y() + bar.get_height()/2,
                f'{val:,.0f}', va='center', fontsize=9)
    ax.set_xlabel(value_col)
    plt.tight_layout()
    return fig

# 3. 분포 비교 (히스토그램 + KDE)
def plot_distribution(series_a, series_b, labels=('A', 'B')):
    fig, ax = plt.subplots(figsize=(10, 5))
    for series, label, color in zip([series_a, series_b], labels,
                                     ['steelblue', 'coral']):
        series.hist(ax=ax, alpha=0.5, label=label,
                    color=color, bins=30, density=True)
        series.plot.kde(ax=ax, color=color, linewidth=2)
    ax.legend()
    plt.tight_layout()
    return fig
```

### 차트에 직접 레이블 추가 (범례 대신)
```python
# 범례 대신 선 끝에 레이블 직접 표시 (더 읽기 쉬움)
fig, ax = plt.subplots(figsize=(12, 5))
for channel, color in zip(['organic', 'paid', 'referral'],
                           ['steelblue', 'coral', 'green']):
    subset = df[df['channel'] == channel]
    ax.plot(subset['date'], subset['value'], color=color, linewidth=2)
    # 마지막 포인트에 레이블
    ax.annotate(channel,
                xy=(subset['date'].iloc[-1], subset['value'].iloc[-1]),
                xytext=(5, 0), textcoords='offset points',
                color=color, fontweight='bold', va='center')
```

## 안티패턴
- 파이차트로 6개 이상 비교 — 슬라이스 크기 구별 불가
- Y축 0 미시작으로 변화 과장 — 독자 오도
- 이중 Y축 — 두 지표 관계를 임의로 암시
- 색상 10가지 이상 — 인지 과부하
- 제목이 "월별 매출 추이" — "3월 매출 전년 대비 12% 성장"으로 작성

## 실전 팁
- 색맹 친화 팔레트: ColorBrewer, Viridis, Tableau 10
- 폰트 크기: 제목 14pt 이상, 레이블 10pt 이상 (프레젠테이션 기준)
- 애니메이션 차트는 트렌드 스토리텔링에만 (데이터 분석에는 정적 차트)
- 차트 해상도: 화면용 72dpi, 인쇄용 300dpi, 발표용 150dpi
- "차트 없이 한 문장으로 설명 가능한가?" — 가능하면 테이블이 더 나을 수도
