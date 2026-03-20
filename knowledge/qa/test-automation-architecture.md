# 테스트 자동화 아키텍처 — 유지보수 가능한 테스트 코드 설계

> 테스트 코드도 코드다. 설계 없이 쌓이면 레거시가 된다.

## 핵심 원칙
- 테스트 코드는 프로덕션 코드와 동일한 설계 원칙을 따른다
- 한 곳의 변경이 수십 개 테스트를 깨뜨리면 아키텍처 문제다
- 추상화는 신중하게: 과한 추상화는 디버깅을 어렵게 한다
- 테스트 인프라 코드는 `src/test-utils/`로 분리한다

## 판단 기준
레이어 구조:
```
tests/
├── unit/          # 빠른 단위 테스트
├── integration/   # DB/API 통합 테스트
├── e2e/           # Playwright E2E
│   ├── pages/     # Page Object Models
│   ├── fixtures/  # 테스트 픽스처
│   └── helpers/   # 공통 헬퍼
└── performance/   # k6 성능 테스트

src/test-utils/
├── factories.ts   # 데이터 팩토리
├── db.ts          # DB 테스트 헬퍼
├── auth.ts        # 인증 헬퍼
└── matchers.ts    # 커스텀 matcher
```

## 코드 예시 (TypeScript/Playwright)
```typescript
// e2e/fixtures/index.ts — Playwright 커스텀 픽스처
import { test as base, expect } from '@playwright/test'
import { LoginPage } from '../pages/LoginPage'
import { DashboardPage } from '../pages/DashboardPage'
import { createTestUserViaApi, deleteTestUserViaApi } from '../helpers/api'

type TestFixtures = {
  loginPage: LoginPage
  dashboardPage: DashboardPage
  authenticatedPage: Page
  testUser: { email: string; password: string; token: string }
}

export const test = base.extend<TestFixtures>({
  testUser: async ({}, use) => {
    const user = await createTestUserViaApi()
    await use(user)
    await deleteTestUserViaApi(user.id)
  },

  authenticatedPage: async ({ page, testUser }, use) => {
    // 스토리지 스테이트로 로그인 상태 주입
    await page.goto('/login')
    await page.getByLabel('이메일').fill(testUser.email)
    await page.getByLabel('비밀번호').fill(testUser.password)
    await page.getByRole('button', { name: '로그인' }).click()
    await page.waitForURL('/dashboard')
    await use(page)
  },

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page))
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page))
  },
})

export { expect }

// 사용 예시
import { test, expect } from '../fixtures'

test('로그인한 사용자는 대시보드에 접근한다', async ({ authenticatedPage }) => {
  await expect(authenticatedPage.getByRole('heading', { name: '대시보드' }))
    .toBeVisible()
})
```

## 안티패턴
- 모든 E2E 테스트 파일에 로그인 코드 복붙
- Helper 함수 없이 모든 로케이터를 테스트 파일에 직접 작성
- 추상화 레이어가 너무 많아 실제 동작 파악이 어려움
- 테스트 인프라 코드에 테스트 없음

## 실전 팁
- POM은 로케이터와 액션만 담고 단언(assertion)은 테스트 파일에 둔다
- 공통 픽스처는 Playwright의 `test.extend`로 타입 안전하게 공유한다
- 테스트 헬퍼에 JSDoc으로 사용 예시를 작성한다
- 팀 온보딩 문서에 테스트 작성 가이드를 포함한다
