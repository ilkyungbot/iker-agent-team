# 리포팅 — 분석 결과를 의사결정으로 연결하기

> 리포트는 데이터를 보여주는 것이 아니라 행동을 촉구하는 것이다.

## 핵심 원칙
- 독자가 리포트를 읽고 무엇을 해야 하는지 알아야 한다
- Bottom Line Up Front (BLUF): 결론을 먼저, 근거는 나중에
- 숫자는 맥락 없이 의미 없다 (전년 대비, 목표 대비, 경쟁사 대비)
- 리포트 주기는 의사결정 주기에 맞춰라
- 자동화 가능한 리포트는 자동화하라 — 사람은 인사이트에 집중

## 판단 기준
| 리포트 유형 | 주기 | 독자 | 핵심 내용 |
|-------------|------|------|-----------|
| 임원 대시보드 | 실시간/일별 | C-Level | KPI 현황, 이상 징후 |
| 주간 성과 리포트 | 주별 | 팀장 | WoW 변화, 이슈, 다음 주 계획 |
| 실험 결과 리포트 | 실험 종료 후 | 실무자+팀장 | 가설, 결과, 권고사항 |
| 심층 분석 리포트 | 비정기 | 의사결정자 | 문제 정의, 분석, 행동 방안 |

## 실전 예시

### 주간 KPI 리포트 SQL 자동화
```sql
-- 주간 KPI 요약 쿼리 (매주 월요일 자동 실행)
WITH current_week AS (
  SELECT
    COUNT(DISTINCT user_id) AS dau_avg,
    SUM(amount) AS weekly_revenue,
    COUNT(DISTINCT CASE WHEN event_name = 'purchase' THEN user_id END) AS buyers,
    COUNT(DISTINCT user_id) AS active_users
  FROM events
  WHERE event_date BETWEEN DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY), WEEK)
    AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
),
prior_week AS (
  SELECT
    COUNT(DISTINCT user_id) AS dau_avg,
    SUM(amount) AS weekly_revenue,
    COUNT(DISTINCT CASE WHEN event_name = 'purchase' THEN user_id END) AS buyers,
    COUNT(DISTINCT user_id) AS active_users
  FROM events
  WHERE event_date BETWEEN DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY), WEEK)
    AND DATE_SUB(CURRENT_DATE(), INTERVAL 8 DAY)
)
SELECT
  c.weekly_revenue,
  ROUND((c.weekly_revenue - p.weekly_revenue) / p.weekly_revenue * 100, 1) AS revenue_wow_pct,
  c.active_users,
  ROUND((c.active_users - p.active_users) / p.active_users * 100, 1) AS users_wow_pct,
  ROUND(c.buyers / c.active_users * 100, 1) AS conversion_rate
FROM current_week c, prior_week p;
```

### Python으로 자동 리포트 생성
```python
import pandas as pd
from datetime import datetime, timedelta
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

def generate_weekly_report(kpi_df: pd.DataFrame) -> str:
    """KPI 데이터프레임을 HTML 리포트로 변환"""
    week_str = datetime.now().strftime('%Y년 %m월 %d일')

    # 변화율에 따른 색상
    def color_change(val):
        if isinstance(val, str) and '%' in val:
            num = float(val.replace('%', ''))
            if num > 0:
                return f'<span style="color:green">▲ {val}</span>'
            elif num < 0:
                return f'<span style="color:red">▼ {val}</span>'
        return val

    html = f"""
    <html><body>
    <h2>📊 주간 KPI 리포트 ({week_str})</h2>
    <h3>💡 핵심 요약</h3>
    <ul>
      <li>주간 매출: {kpi_df['revenue'].iloc[0]:,.0f}원
          ({color_change(f"{kpi_df['revenue_wow']:.1f}%")} WoW)</li>
      <li>활성 사용자: {kpi_df['active_users'].iloc[0]:,}명</li>
      <li>전환율: {kpi_df['conversion_rate'].iloc[0]:.1f}%</li>
    </ul>
    <h3>🔍 주요 이슈</h3>
    <p>[분석 담당자가 작성]</p>
    <h3>📌 다음 주 집중 사항</h3>
    <p>[팀 논의 후 작성]</p>
    </body></html>
    """
    return html
```

### 리포트 구조 템플릿
```
# 주간 비즈니스 리포트 (YYYY-WW)

## 한 줄 요약
"이번 주 매출은 X원으로 목표 대비 Y% [달성/미달]했으며,
주요 원인은 Z입니다."

## 핵심 지표
| 지표 | 이번 주 | 지난 주 | WoW | 목표 | 달성률 |
|------|---------|---------|-----|------|--------|

## 주목할 변화 (Top 3)
1. [현상] → [원인 가설] → [필요 조치]
2.
3.

## 실험/이니셔티브 현황
- 실험 A: 결과 / 권고사항

## 다음 주 집중 사항
- [ ] 액션 아이템
```

## 안티패턴
- 숫자 나열만 하는 리포트 — "그래서?" 질문에 답 없음
- 모든 지표를 동등하게 나열 — 우선순위 없음
- 부정적인 지표 숨기기 또는 작게 표시
- 리포트마다 지표 정의가 달라지는 경우
- 독자가 알아서 인사이트를 찾을 것이라는 기대

## 실전 팁
- 리포트 before/after: "데이터 → 인사이트 → 액션" 3단 구조
- 예외 리포팅: 이상할 때만 알리면 신뢰도 올라감
- 리포트 리뷰 미팅: 숫자 읽기 금지, 인사이트와 결정만
- 1페이저 원칙: 핵심 리포트는 A4 1장에 담아라
- 자동화 우선순위: 매주 반복되는 리포트부터 자동화
