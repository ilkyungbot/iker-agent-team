# CI/CD — 테스트를 자동으로 실행하는 파이프라인 설계

> CI는 팀의 품질 게이트다. 느리거나 신뢰할 수 없으면 아무도 기다리지 않는다.

## 핵심 원칙
- 모든 PR은 CI 통과 없이 머지할 수 없다
- 전체 파이프라인은 10분 이내 완료를 목표로 한다
- 실패 원인은 명확하게 리포팅한다 (링크, 스크린샷, 로그)
- 환경 변수와 시크릿은 CI 시크릿 스토어에서 관리한다

## 판단 기준
파이프라인 단계 구성:
1. **Lint/Type Check**: 1-2분 (빠른 피드백)
2. **단위 테스트**: 2-3분 (병렬 실행)
3. **통합 테스트**: 3-5분 (DB 컨테이너 포함)
4. **E2E 테스트**: 5-10분 (샤딩으로 병렬화)
5. **빌드 + 배포**: 스테이징 환경

트리거 전략:
- PR 오픈/업데이트: 전체 파이프라인
- main 머지: 전체 파이프라인 + 배포
- 스케줄(야간): E2E + 성능 테스트

## 코드 예시 (GitHub Actions)
```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  unit-tests:
    runs-on: ubuntu-latest
    needs: lint-and-type-check
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v4

  e2e-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run test:e2e -- --shard=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report-${{ matrix.shard }}
          path: playwright-report/
```

## 안티패턴
- CI 실패를 무시하고 force merge
- 모든 테스트를 직렬로 실행해 30분 이상 소요
- 시크릿을 코드에 하드코딩
- 플레이키 테스트를 `retry`로 덮어 근본 원인 방치

## 실전 팁
- `actions/cache`로 `node_modules`, Playwright 바이너리를 캐싱한다
- 실패 시 `upload-artifact`로 스크린샷/로그를 첨부한다
- 커버리지 리포트는 PR 코멘트로 자동 게시한다
- 야간 E2E는 실패 시 Slack 알림을 자동으로 보낸다
