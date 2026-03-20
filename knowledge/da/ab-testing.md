# A/B 테스팅 — 올바른 실험 설계와 해석

> 실험은 직관을 검증하는 도구다. 설계 없는 실험은 데이터 낭비다.

## 핵심 원칙
- 가설 먼저: "X를 바꾸면 Y가 Z% 개선될 것이다"
- 샘플 사이즈를 사전에 계산하라 (사후 계산은 의미 없음)
- 하나의 실험에 하나의 변수만 바꿔라
- 실험 기간은 최소 1-2주 (요일 효과 제거)
- 실험 시작 전 SRM(샘플 비율 불일치) 체크

## 판단 기준
| 기준 | 권장값 | 이유 |
|------|--------|------|
| 유의수준 (α) | 0.05 | 5% 오류 허용 |
| 검정력 (1-β) | 0.80 | 20% 2종 오류 허용 |
| 최소 탐지 효과 (MDE) | 비즈니스 가치 기준 설정 | |
| 실험 기간 | 완전한 주 단위 | 요일 편향 제거 |

## 실전 예시

### 샘플 사이즈 계산 (Python)
```python
from scipy import stats
import numpy as np

def calculate_sample_size(baseline_rate, mde, alpha=0.05, power=0.80):
    """
    baseline_rate: 기존 전환율 (예: 0.05 = 5%)
    mde: 최소 탐지 효과 (예: 0.01 = 1%p 개선)
    """
    p1 = baseline_rate
    p2 = baseline_rate + mde

    z_alpha = stats.norm.ppf(1 - alpha/2)
    z_beta = stats.norm.ppf(power)

    pooled_p = (p1 + p2) / 2
    n = (z_alpha * np.sqrt(2 * pooled_p * (1 - pooled_p)) +
         z_beta * np.sqrt(p1 * (1-p1) + p2 * (1-p2)))**2 / (p1 - p2)**2

    return int(np.ceil(n))

# 예: 기존 전환율 5%, 1%p 개선 목표
n = calculate_sample_size(0.05, 0.01)
print(f"그룹당 필요 샘플: {n:,}명")
```

### 결과 분석 (카이제곱 검정)
```python
from scipy.stats import chi2_contingency, fisher_exact
import pandas as pd

def analyze_ab_test(control_visitors, control_conversions,
                    treatment_visitors, treatment_conversions):
    control_non = control_visitors - control_conversions
    treatment_non = treatment_visitors - treatment_conversions

    contingency_table = [
        [control_conversions, control_non],
        [treatment_conversions, treatment_non]
    ]

    chi2, p_value, dof, expected = chi2_contingency(contingency_table)

    control_rate = control_conversions / control_visitors
    treatment_rate = treatment_conversions / treatment_visitors
    lift = (treatment_rate - control_rate) / control_rate * 100

    print(f"대조군 전환율: {control_rate:.2%}")
    print(f"실험군 전환율: {treatment_rate:.2%}")
    print(f"상대적 개선: {lift:.1f}%")
    print(f"p-value: {p_value:.4f}")
    print(f"유의함: {'Yes' if p_value < 0.05 else 'No'}")

    return {'p_value': p_value, 'lift': lift, 'significant': p_value < 0.05}

analyze_ab_test(10000, 480, 10000, 530)
```

### SRM 체크 (샘플 비율 불일치)
```sql
-- 실험군/대조군 비율이 의도한 50:50인지 확인
SELECT
  variant,
  COUNT(*) AS user_count,
  COUNT(*) / SUM(COUNT(*)) OVER () AS actual_ratio
FROM experiment_assignments
WHERE experiment_id = 'exp_001'
GROUP BY variant;
-- actual_ratio가 0.45~0.55 벗어나면 SRM 의심
```

## 안티패턴
- Peeking: 중간에 p-value 보고 유의하면 일찍 종료 — 1종 오류 급증
- 다중 검정 문제: 지표 10개 동시에 보면 우연히 유의한 것 나옴 → Bonferroni 보정
- 신뢰기간 무시하고 단 하루 결과로 결론
- SRM 미확인 — 실험 배정 버그 있어도 모름
- 실험 후 세그먼트 분쪼개기 (p-hacking)

## 실전 팁
- 실험 시작 전 Pre-registration: 가설, 지표, 샘플 사이즈, 기간 문서화
- Novelty Effect: 새 기능은 초반에 일시적으로 좋게 보임 — 2-3주 관찰
- AA 테스트로 실험 인프라 먼저 검증
- 실용적 유의성(Effect Size)과 통계적 유의성을 함께 봐라
- 베이지안 방법: 사전 지식 활용, 연속 모니터링 가능 (단, 해석 복잡)
