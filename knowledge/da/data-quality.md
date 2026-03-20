# 데이터 품질 — 신뢰할 수 있는 데이터를 만드는 방법

> 나쁜 데이터로 분석하느니 분석 안 하는 게 낫다. 데이터 품질은 인프라다.

## 핵심 원칙
- 데이터 품질 문제는 소스에서 해결해야 한다. 분석 단계에서는 이미 늦다
- 품질 체크는 파이프라인에 자동화하라 — 사람이 매번 확인하면 실패한다
- 이상 징후를 발견하면 먼저 데이터 문제인지 실제 현상인지 구분하라
- "완벽한 데이터"를 기다리지 말고, 현재 품질 수준을 문서화하고 분석하라

## 판단 기준
| 품질 차원 | 정의 | 체크 방법 |
|-----------|------|-----------|
| 완전성(Completeness) | NULL/결측값 비율 | NULL count / 전체 행 |
| 정확성(Accuracy) | 실제 값과 일치 여부 | 참조 데이터와 비교 |
| 일관성(Consistency) | 시스템 간 동일 여부 | 크로스 체크 |
| 시의성(Timeliness) | 최신 데이터 여부 | max(updated_at) |
| 유일성(Uniqueness) | 중복 레코드 여부 | COUNT vs COUNT DISTINCT |

## 실전 예시

### SQL 데이터 품질 체크
```sql
-- 종합 품질 리포트
WITH quality_check AS (
  SELECT
    COUNT(*) AS total_rows,
    COUNT(user_id) AS non_null_user_id,
    COUNT(DISTINCT user_id) AS unique_users,
    COUNTIF(user_id IS NULL) AS null_user_id,
    COUNTIF(amount < 0) AS negative_amount,
    COUNTIF(event_date > CURRENT_DATE()) AS future_dates,
    MIN(event_date) AS earliest_date,
    MAX(event_date) AS latest_date,
    DATE_DIFF(CURRENT_DATE(), MAX(event_date), DAY) AS days_since_last_record
  FROM events
  WHERE event_date = CURRENT_DATE() - 1  -- 어제 데이터 체크
)
SELECT
  *,
  ROUND(null_user_id / total_rows * 100, 2) AS null_rate_pct,
  ROUND((total_rows - unique_users) / total_rows * 100, 2) AS duplicate_rate_pct,
  CASE
    WHEN null_user_id / total_rows > 0.01 THEN 'FAIL: NULL 1% 초과'
    WHEN negative_amount > 0 THEN 'FAIL: 음수 금액 존재'
    WHEN future_dates > 0 THEN 'FAIL: 미래 날짜 존재'
    WHEN days_since_last_record > 1 THEN 'WARN: 최신 데이터 없음'
    ELSE 'PASS'
  END AS quality_status
FROM quality_check;
```

### Python Great Expectations 스타일 검증
```python
import pandas as pd
import numpy as np
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class QualityRule:
    name: str
    check_fn: callable
    severity: str  # 'error' or 'warning'

def run_quality_checks(df: pd.DataFrame,
                       rules: List[QualityRule]) -> dict:
    results = {'passed': [], 'failed': [], 'warnings': []}

    for rule in rules:
        passed, message = rule.check_fn(df)
        if passed:
            results['passed'].append(rule.name)
        elif rule.severity == 'error':
            results['failed'].append(f"{rule.name}: {message}")
        else:
            results['warnings'].append(f"{rule.name}: {message}")

    return results

# 규칙 정의
rules = [
    QualityRule(
        name="user_id_not_null",
        check_fn=lambda df: (
            df['user_id'].notna().all(),
            f"NULL {df['user_id'].isna().sum()}건"
        ),
        severity='error'
    ),
    QualityRule(
        name="amount_positive",
        check_fn=lambda df: (
            (df['amount'] >= 0).all(),
            f"음수 {(df['amount'] < 0).sum()}건"
        ),
        severity='error'
    ),
    QualityRule(
        name="duplicate_check",
        check_fn=lambda df: (
            df.duplicated(subset=['order_id']).sum() == 0,
            f"중복 {df.duplicated(subset=['order_id']).sum()}건"
        ),
        severity='warning'
    ),
]

# 볼륨 이상 탐지 (Z-score)
def detect_volume_anomaly(daily_counts: pd.Series,
                          threshold_sigma: float = 3.0) -> bool:
    mean = daily_counts.mean()
    std = daily_counts.std()
    latest = daily_counts.iloc[-1]
    z_score = abs(latest - mean) / std
    return z_score > threshold_sigma, f"Z-score: {z_score:.2f}"
```

## 안티패턴
- 품질 체크 없이 파이프라인 운영 — 오염 데이터 자동 확산
- NULL을 0으로 채우기 — NULL과 0은 의미가 다름
- 중복 레코드 방치 — 집계 결과 뻥튀기
- 품질 이슈 발견 시 데이터 삭제 — 원인 파악 전에 증거 소멸
- 모든 필드를 VARCHAR로 저장 — 타입 강제가 품질 보장

## 실전 팁
- dbt test: not_null, unique, accepted_values, relationships 4가지 기본 테스트
- 볼륨 모니터링: 전일 대비 ±30% 이상 변동 시 알림
- 데이터 품질 SLA: "일별 결측률 1% 이하" 같은 기준 수치화
- 품질 대시보드: 소스 시스템별 품질 지표 한눈에
- 이상 탐지 자동화 후 → 인간은 원인 분석에 집중
