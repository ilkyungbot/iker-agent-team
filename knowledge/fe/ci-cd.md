# CI/CD — 지속적 통합/배포

> 자동화는 실수를 줄이고, 빠른 피드백은 품질을 높인다.

## 핵심 원칙
- **빠른 피드백**: PR 열면 5분 내 기본 검사 완료
- **자동화 범위**: lint, type-check, test, build 모두 자동
- **프리뷰 배포**: PR마다 독립 환경에서 확인
- **main 항상 배포 가능**: 브랜치 전략으로 안정성 보장

## 브랜치 전략 (Trunk-based 변형)
```
main          → 프로덕션 배포
  └── feature/xxx  → PR → main (빠르게 머지)
  └── hotfix/xxx   → PR → main (긴급 수정)
```

## 코드 예시

✅ GitHub Actions CI
```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run type-check

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm run test -- --coverage

      - name: Build
        run: npm run build
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

✅ Pre-commit 훅 (Husky + lint-staged)
```json
// package.json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,css}": "prettier --write"
  }
}
```

```bash
# .husky/pre-commit
#!/bin/sh
npx lint-staged
```

✅ Vercel 배포 설정
```json
// vercel.json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "env": {
    "NEXT_PUBLIC_API_URL": "@api-url"
  },
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" }
      ]
    }
  ]
}
```

✅ package.json 스크립트 표준화
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "analyze": "ANALYZE=true next build",
    "prepare": "husky"
  }
}
```

## 실전 팁
- `npm ci` vs `npm install` — CI에서는 `npm ci`로 lock 파일 엄수
- Secrets는 GitHub Secrets에 저장, 코드에 하드코딩 금지
- 빌드 캐시 (`.next/cache`) CI에서 캐싱으로 속도 향상

## 주의사항
- 테스트 없이 main 머지 금지 — branch protection rule 설정
- E2E 테스트를 CI 기본에 넣으면 느림 — 별도 스케줄 또는 merge 후 실행
- `npm run build` 로컬에서 통과해도 CI에서 실패할 수 있음 — 환경 변수 차이
