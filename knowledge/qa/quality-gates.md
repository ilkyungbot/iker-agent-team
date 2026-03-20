# 품질 게이트 — 기준 이하는 통과시키지 않는다

> 품질 게이트 없는 CI는 자동화된 고무 도장이다.

## 핵심 원칙
- 게이트는 팀이 합의한 기준이어야 한다 (일방적 강요 금지)
- 숫자보다 트렌드가 중요하다 (커버리지 80% 유지 vs 낮아지는 추세)
- 새 코드에 더 엄격한 기준을 적용한다 (Ratchet 패턴)
- 게이트 실패는 명확한 원인과 해결 방법을 제시해야 한다

## 판단 기준
| 게이트 항목 | 최소 기준 | 권장 기준 |
|------------|---------|---------|
| 라인 커버리지 | 70% | 85% |
| 브랜치 커버리지 | 65% | 80% |
| 중복 코드 | < 5% | < 3% |
| 복잡도 (함수당) | < 15 | < 10 |
| 빌드 시간 | < 15분 | < 10분 |
| 취약점 (Critical) | 0개 | 0개 |
| TypeScript 에러 | 0개 | 0개 |

## 코드 예시 (TypeScript/Vitest)
```typescript
// vitest.config.ts — 커버리지 게이트
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      thresholds: {
        global: {
          branches: 80,
          functions: 80,
          lines: 80,
          statements: 80,
        },
        // 특정 경로에 더 높은 기준 적용
        'src/payment/**': {
          branches: 95,
          functions: 95,
          lines: 95,
        },
      },
    },
  },
})

// .github/workflows/quality-gate.yml 핵심
// SonarCloud 품질 게이트 예시
- name: SonarCloud Scan
  uses: SonarSource/sonarcloud-github-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

// package.json 스크립트
{
  "scripts": {
    "test:ci": "vitest run --coverage",
    "type-check": "tsc --noEmit",
    "lint": "eslint src --max-warnings 0",
    "quality-check": "npm run type-check && npm run lint && npm run test:ci"
  }
}
```

## 안티패턴
- 게이트를 통과하기 위해 의미 없는 테스트 추가
- 팀 전체 합의 없이 게이트 기준을 낮춤
- Critical 취약점을 `ignore` 처리하고 게이트 통과
- 신규 코드와 레거시 코드에 동일한 기준 강요

## 실전 팁
- SonarCloud/SonarQube 무료 플랜으로 코드 품질 대시보드를 만든다
- PR별 커버리지 변화를 코멘트로 자동 게시한다 (`codecov`)
- `eslint --max-warnings 0` 으로 경고를 에러로 취급한다
- 게이트 기준을 팀 위키에 문서화하고 정기적으로 리뷰한다
