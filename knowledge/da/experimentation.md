# 실험 플랫폼 — 지속적인 실험 문화 구축

> 한 번의 실험보다 실험할 수 있는 문화가 중요하다. 실험 속도가 성장 속도다.

## 핵심 원칙
- 실험은 배움을 위한 것이다 — "실패"한 실험도 가치 있다
- 실험 문화: 직관이 아닌 데이터로 의사결정하는 문화
- 실험 인프라가 없으면 실험 속도가 나오지 않는다
- 동시에 여러 실험 실행 시 상호 작용(Interaction Effect) 주의
- 실험 결과는 문서화하고 조직 지식으로 축적하라

## 판단 기준
| 실험 유형 | 설명 | 적합한 상황 |
|-----------|------|-------------|
| A/B Test | 두 버전 비교 | UI 변경, 카피 최적화 |
| Multivariate Test | 여러 변수 동시 비교 | 복합 페이지 최적화 |
| Holdout Test | 기능 그룹 vs 비기능 그룹 | 기능 전체 효과 측정 |
| Switchback Test | 시간대별 교차 | 마켓플레이스, 실시간 서비스 |
| Bandit Test | 동적 트래픽 배분 | 빠른 최적화 필요 시 |

## 실전 예시

### 실험 추적 테이블 설계
```sql
-- 실험 메타데이터
CREATE TABLE experiments (
  experiment_id STRING,
  experiment_name STRING,
  hypothesis STRING,
  primary_metric STRING,
  guardrail_metrics ARRAY<STRING>,
  start_date DATE,
  end_date DATE,
  target_sample_size INT64,
  traffic_allocation FLOAT64,  -- 0.5 = 50%
  status STRING,  -- 'running', 'completed', 'stopped'
  owner STRING,
  created_at TIMESTAMP
);

-- 실험 배정 로그
CREATE TABLE experiment_assignments (
  user_id STRING,
  experiment_id STRING,
  variant STRING,  -- 'control', 'treatment', 'treatment_a', 'treatment_b'
  assigned_at TIMESTAMP,
  PRIMARY KEY (user_id, experiment_id)
);
```

### 실험 결과 분석 자동화
```python
from scipy import stats
import pandas as pd
import numpy as np

class ExperimentAnalyzer:
    def __init__(self, experiment_id: str, db_client):
        self.experiment_id = experiment_id
        self.db = db_client

    def get_experiment_data(self):
        """실험 배정 + 전환 데이터 조회"""
        query = f"""
        SELECT
          a.user_id,
          a.variant,
          a.assigned_at,
          COUNTIF(e.event_name = 'purchase') > 0 AS converted,
          SUM(CASE WHEN e.event_name = 'purchase' THEN e.amount ELSE 0 END) AS revenue
        FROM experiment_assignments a
        LEFT JOIN events e
          ON a.user_id = e.user_id
          AND e.event_date BETWEEN DATE(a.assigned_at) AND DATE_ADD(DATE(a.assigned_at), INTERVAL 7 DAY)
        WHERE a.experiment_id = '{self.experiment_id}'
        GROUP BY 1, 2, 3
        """
        return self.db.query(query)

    def check_srm(self, df, expected_ratio=0.5):
        """샘플 비율 불일치(SRM) 검사"""
        observed = df['variant'].value_counts()
        total = len(df)
        expected = {v: total * expected_ratio for v in observed.index}

        chi2, p_value = stats.chisquare(
            f_obs=list(observed.values),
            f_exp=list(expected.values())
        )
        if p_value < 0.01:
            print(f"경고: SRM 감지됨 (p={p_value:.4f})")
        return p_value

    def analyze_conversion(self, df):
        """전환율 분석"""
        results = []
        control = df[df['variant'] == 'control']

        for variant in df['variant'].unique():
            if variant == 'control':
                continue
            treatment = df[df['variant'] == variant]

            # 전환율
            c_rate = control['converted'].mean()
            t_rate = treatment['converted'].mean()
            lift = (t_rate - c_rate) / c_rate * 100

            # 통계 검정
            contingency = [
                [control['converted'].sum(), (~control['converted']).sum()],
                [treatment['converted'].sum(), (~treatment['converted']).sum()]
            ]
            _, p_value, _, _ = stats.chi2_contingency(contingency)

            results.append({
                'variant': variant,
                'n_control': len(control),
                'n_treatment': len(treatment),
                'control_rate': f"{c_rate:.2%}",
                'treatment_rate': f"{t_rate:.2%}",
                'lift': f"{lift:+.1f}%",
                'p_value': round(p_value, 4),
                'significant': p_value < 0.05
            })

        return pd.DataFrame(results)
```

### 실험 문서 템플릿
```markdown
## 실험: [실험 이름]

### 가설
"[변경 사항]을 하면 [대상 사용자]의 [지표]가 [방향]할 것이다.
왜냐하면 [근거]이기 때문이다."

### 설계
- 대조군: 현재 경험
- 실험군: 변경된 경험
- 트래픽: 50:50
- 기간: 2주 (2024-03-01 ~ 2024-03-14)
- 필요 샘플: 그룹당 5,000명

### 주요 지표
- Primary: 7일 전환율
- Guardrail: 페이지 로딩 시간, 오류율

### 결과
- SRM: Pass
- 전환율: 대조군 4.8% vs 실험군 5.4% (+12.5%, p=0.023)
- 권고: 실험군 전체 배포
```

## 안티패턴
- 실험 중간에 결과 보고 조기 종료 (Peeking Problem)
- 실패한 실험 결과를 문서화하지 않음 — 조직 학습 손실
- 실험 없이 "데이터 분석 결과"만으로 기능 효과 주장
- 동시에 너무 많은 실험 — 상호작용 효과 측정 불가
- 실험 인프라 없이 코드 플래그로만 관리 — 배정 로그 없음

## 실전 팁
- 실험 로드맵: 우선순위 실험 목록 + 예상 임팩트
- 실험 리뷰 미팅: 주간 실험 결과 공유 + 다음 실험 논의
- 실험 가속화: 트래픽이 적을 때 → 다중 실험 동시 진행
- 실험 결과 DB: 모든 실험 가설-결과-학습 축적 → 패턴 발견
- 실험 민주화: 엔지니어 없이 마케팅/PM이 직접 실험 가능한 도구
