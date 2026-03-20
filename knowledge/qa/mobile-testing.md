# 모바일 테스트 — 데스크톱과 다른 환경을 고려한다

> 모바일 사용자는 다른 네트워크, 다른 해상도, 다른 입력 방식을 사용한다.

## 핵심 원칙
- 실제 디바이스 테스트가 에뮬레이터보다 우선이다
- 네트워크 조건을 다양화한다 (3G, 오프라인)
- 터치 이벤트와 제스처를 별도로 검증한다
- iOS와 Android는 렌더링과 동작이 다를 수 있다

## 판단 기준
테스트 우선순위 디바이스:
- 최신 iPhone (Safari)
- 최신 Android (Chrome)
- 중저가 Android (메모리/성능 제약)
- 태블릿 (레이아웃 전환)

모바일 전용 테스트 항목:
- [ ] 터치 영역 최소 44x44px
- [ ] 가로/세로 모드 전환
- [ ] 키보드 올라올 때 레이아웃 밀림
- [ ] 핀치 줌 동작
- [ ] 스크롤 성능 (60fps)
- [ ] 오프라인/느린 네트워크 UX

## 코드 예시 (Playwright 모바일)
```typescript
// playwright.config.ts — 모바일 프로젝트 설정
import { devices } from '@playwright/test'

export default defineConfig({
  projects: [
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 14'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 7'] },
    },
    {
      name: 'Slow Network',
      use: {
        ...devices['iPhone 14'],
        // 3G 네트워크 시뮬레이션
        launchOptions: {
          args: ['--force-fieldtrials=NetworkQualityEstimator/Enabled'],
        },
      },
    },
  ],
})

// e2e/mobile.spec.ts
test('모바일에서 터치로 메뉴를 열 수 있다', async ({ page }) => {
  await page.goto('/')

  // 햄버거 메뉴 탭
  await page.getByRole('button', { name: '메뉴 열기' }).tap()
  await expect(page.getByRole('navigation')).toBeVisible()
})

test('3G 환경에서도 3초 내 핵심 콘텐츠가 표시된다', async ({ page, context }) => {
  await context.route('**/*', async (route) => {
    await new Promise(resolve => setTimeout(resolve, 200)) // 3G 지연 시뮬레이션
    await route.continue()
  })

  await page.goto('/')
  await expect(page.getByRole('main')).toBeVisible({ timeout: 3000 })
})
```

## 안티패턴
- 데스크톱에서만 테스트하고 "모바일도 같겠지" 가정
- 에뮬레이터만 사용하고 실제 디바이스 미검증
- 모바일 성능 최적화를 배포 후로 미루기
- `hover` 이벤트에만 의존하는 인터랙션 설계

## 실전 팁
- BrowserStack이나 LambdaTest로 다양한 실제 디바이스에서 테스트한다
- Chrome DevTools의 네트워크 스로틀링으로 빠른 로컬 검증이 가능하다
- iOS 사파리 전용 버그는 별도 태그(`@safari`)로 추적한다
- 모바일 퍼스트 CSS로 반응형을 구현하면 테스트도 단순해진다
