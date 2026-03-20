# CI/CD

## GitHub Actions 워크플로우

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '22'
  DATABASE_URL: postgresql://postgres:password@localhost:5432/testdb
  REDIS_URL:    redis://localhost:6379

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB:       testdb
          POSTGRES_USER:     postgres
          POSTGRES_PASSWORD: password
        ports:  ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:    ['6379:6379']
        options:  --health-cmd "redis-cli ping" --health-interval 10s

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type Check
        run: npm run type-check

      - name: Run Migrations
        run: npx drizzle-kit migrate

      - name: Test
        run: npm run test:coverage

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

## 배포 파이프라인

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker Image
        run: |
          docker build -t ${{ secrets.REGISTRY }}/api:${{ github.sha }} .
          docker tag ${{ secrets.REGISTRY }}/api:${{ github.sha }} \
                     ${{ secrets.REGISTRY }}/api:latest

      - name: Push to Registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push ${{ secrets.REGISTRY }}/api:${{ github.sha }}
          docker push ${{ secrets.REGISTRY }}/api:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/api-server \
            api-server=${{ secrets.REGISTRY }}/api:${{ github.sha }}
          kubectl rollout status deployment/api-server --timeout=300s
```

## package.json 스크립트

```json
{
  "scripts": {
    "dev":          "tsx watch src/server.ts",
    "build":        "tsc -p tsconfig.json",
    "start":        "node dist/server.js",
    "test":         "vitest run",
    "test:watch":   "vitest",
    "test:coverage":"vitest run --coverage",
    "lint":         "eslint src --ext .ts",
    "lint:fix":     "eslint src --ext .ts --fix",
    "type-check":   "tsc --noEmit",
    "db:generate":  "drizzle-kit generate",
    "db:migrate":   "drizzle-kit migrate",
    "db:studio":    "drizzle-kit studio"
  }
}
```

## Husky + lint-staged (커밋 전 자동 검사)

```json
// .husky/pre-commit
// package.json
{
  "lint-staged": {
    "src/**/*.ts": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

```bash
# 설정
npm install -D husky lint-staged
npx husky init
echo "npx lint-staged" > .husky/pre-commit
```

## 브랜치 전략

```
main          ← 프로덕션
  └─ develop  ← 스테이징
       └─ feature/xxx  ← 기능 개발
       └─ fix/xxx      ← 버그 수정
       └─ hotfix/xxx   ← 긴급 수정 (main으로 직접 머지)
```

## 체크리스트
- [ ] PR마다 lint + type-check + test 실행
- [ ] 테스트 통과 없이 main 머지 차단
- [ ] 도커 이미지 태그에 git sha 사용
- [ ] 배포 후 롤백 절차 문서화
- [ ] 시크릿은 GitHub Secrets에 저장
