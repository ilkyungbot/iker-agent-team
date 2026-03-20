# Playwright — 현대적인 E2E 자동화 표준

> Playwright는 Cypress보다 빠르고, Selenium보다 안정적이다. 올바르게 설정하면 강력하다.

## 핵심 원칙
- 로케이터는 접근성 속성(`role`, `label`, `text`) 우선, `data-testid` 차선
- Auto-wait를 믿어라. 명시적 `sleep`은 절대 사용하지 않는다
- Page Object Model(POM)로 로케이터와 액션을 캡슐화한다
- 병렬 실행과 샤딩으로 CI 속도를 최적화한다

## 판단 기준
로케이터 선택 순서:
1. `getByRole` (접근성 시맨틱)
2. `getByLabel` (폼 요소)
3. `getByText` (사용자 눈에 보이는 텍스트)
4. `getByTestId` (위 방법이 불가한 경우)
5. `locator('.css-selector')` (최후 수단)

## 코드 예시 (Playwright)
```typescript
// e2e/pages/LoginPage.ts — Page Object Model
export class LoginPage {
  constructor(private readonly page: Page) {}

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('이메일').fill(email)
    await this.page.getByLabel('비밀번호').fill(password)
    await this.page.getByRole('button', { name: '로그인' }).click()
    await this.page.waitForURL('/dashboard')
  }

  async expectError(message: string) {
    await expect(this.page.getByRole('alert')).toContainText(message)
  }
}

// e2e/login.spec.ts
import { test, expect } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'

test('잘못된 비밀번호로 로그인 시 에러를 표시한다', async ({ page }) => {
  const loginPage = new LoginPage(page)
  await loginPage.goto()
  await loginPage.login('user@test.com', 'wrongpassword')
  await loginPage.expectError('이메일 또는 비밀번호가 올바르지 않습니다')
})

// playwright.config.ts 핵심 설정
export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: [['html'], ['github']],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
})
```

## 안티패턴
- `page.waitForTimeout(3000)` 사용 (명시적 대기 조건 사용)
- CSS 클래스 선택자에 의존 (UI 리팩터링에 깨짐)
- 테스트 파일 하나에 모든 시나리오 몰아넣기
- `test.only`를 커밋에 포함 (CI에서 다른 테스트가 건너뜀)

## 실전 팁
- `npx playwright codegen` 으로 빠르게 첫 스크립트를 생성한다
- `--ui` 모드로 디버깅 시 실시간 로케이터를 확인한다
- `storageState`로 로그인 상태를 저장해 매 테스트마다 로그인 반복을 피한다
- CI에서 `--shard=1/4` 플래그로 병렬 분산 실행한다
