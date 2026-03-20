# QA 메트릭 — 숫자로 품질을 관리한다

> 측정하지 않으면 개선할 수 없다. 하지만 측정 자체가 목표가 되면 안 된다.

## 핵심 원칙
- 메트릭은 행동을 유도해야 한다. 숫자만 있는 지표는 의미 없다
- 한 가지 메트릭만 최적화하면 다른 것이 희생된다
- 트렌드가 절대값보다 중요하다
- 팀이 동의하는 기준을 먼저 정의한다

## 판단 기준
**핵심 QA 메트릭:**
| 메트릭 | 측정 방법 | 목표 |
|--------|---------|------|
| 테스트 커버리지 | 라인/브랜치/함수 | > 80% |
| 빌드 성공률 | (성공/전체) × 100 | > 95% |
| 테스트 실행 시간 | CI 파이프라인 시간 | < 10분 |
| Flaky 테스트 비율 | 불안정 테스트/전체 | < 1% |
| 버그 탈출률 | 프로덕션 버그/전체 버그 | < 10% |
| MTTR | 인시던트 감지~복구 | < 1시간 |
| 결함 밀도 | 버그/코드 천 줄 | < 1 |

## 코드 예시 (메트릭 수집 및 리포팅)
```typescript
// scripts/collect-test-metrics.ts
import { readFileSync } from 'fs'

interface TestMetrics {
  totalTests: number
  passedTests: number
  failedTests: number
  skippedTests: number
  duration: number
  coverage: {
    lines: number
    branches: number
    functions: number
  }
}

function parseVitestReport(reportPath: string): TestMetrics {
  const report = JSON.parse(readFileSync(reportPath, 'utf-8'))
  return {
    totalTests: report.numTotalTests,
    passedTests: report.numPassedTests,
    failedTests: report.numFailedTests,
    skippedTests: report.numPendingTests,
    duration: report.startTime,
    coverage: report.coverageMap?.summary ?? { lines: 0, branches: 0, functions: 0 },
  }
}

function generateMetricsSummary(metrics: TestMetrics): string {
  const passRate = ((metrics.passedTests / metrics.totalTests) * 100).toFixed(1)
  return [
    `테스트 통과율: ${passRate}% (${metrics.passedTests}/${metrics.totalTests})`,
    `커버리지: 라인 ${metrics.coverage.lines}%, 브랜치 ${metrics.coverage.branches}%`,
    `실행 시간: ${(metrics.duration / 1000).toFixed(1)}초`,
  ].join('\n')
}

// GitHub Actions에서 메트릭을 PR 코멘트로 게시
// .github/workflows/metrics.yml 참고
```

## 안티패턴
- 커버리지 100%를 목표로 의미 없는 테스트 추가
- 메트릭 수집만 하고 액션 없음
- 팀에 공유하지 않고 QA만 보는 지표
- 단일 메트릭(예: 커버리지)만으로 품질 평가

## 실전 팁
- 월간 QA 메트릭 리뷰 미팅을 팀 일정에 고정한다
- 버그 탈출률이 높아지면 테스트 전략을 재검토한다
- 대시보드(Grafana, Datadog)에 메트릭을 시각화한다
- 새 기능 릴리즈 전후의 메트릭 변화를 비교한다
