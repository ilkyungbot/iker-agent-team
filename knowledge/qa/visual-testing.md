# 비주얼 테스트 — 눈에 보이는 회귀를 자동으로 잡는다

> CSS 한 줄 변경이 전혀 다른 페이지의 레이아웃을 깨뜨릴 수 있다. 비주얼 테스트가 잡는다.

## 핵심 원칙
- 스크린샷 비교로 의도치 않은 UI 변경을 자동 탐지한다
- 기준 이미지(baseline)는 버전 관리에 포함한다
- 의도적 변경 시 baseline을 명시적으로 업데이트한다
- 퍼셀 단위 비교보다 시각적 차이 임계값을 활용한다

## 판단 기준
비주얼 테스트 적용 우선 대상:
- 공유 컴포넌트 라이브러리 (버튼, 카드, 모달)
- 마케팅/랜딩 페이지 (레이아웃 중요)
- 대시보드 차트/그래프
- 다크모드/라이트모드 전환

비주얼 테스트 적용 제외:
- 동적 데이터가 많은 페이지 (날짜, 사용자 수)
- 애니메이션이 있는 요소
- 테스트마다 변하는 광고/배너

## 코드 예시 (Playwright 스냅샷)
```typescript
// e2e/visual/components.spec.ts
import { test, expect } from '@playwright/test'

test.describe('비주얼 회귀 테스트', () => {
  test('버튼 컴포넌트가 기준 이미지와 일치한다', async ({ page }) => {
    await page.goto('/storybook/components/button')

    // 전체 페이지 스냅샷
    await expect(page).toHaveScreenshot('button-variants.png', {
      maxDiffPixelRatio: 0.02,  // 2% 이내 차이 허용
      animations: 'disabled',   // 애니메이션 비활성화
    })
  })

  test('모바일에서 네비게이션 레이아웃이 정상이다', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 812 })
    await page.goto('/')

    // 특정 요소만 스냅샷
    const nav = page.locator('nav')
    await expect(nav).toHaveScreenshot('mobile-nav.png')
  })

  test('다크모드 전환 시 색상이 올바르다', async ({ page }) => {
    await page.emulateMedia({ colorScheme: 'dark' })
    await page.goto('/dashboard')

    await expect(page).toHaveScreenshot('dashboard-dark.png', {
      clip: { x: 0, y: 0, width: 1280, height: 720 },
    })
  })
})

// 기준 이미지 업데이트 명령어:
// npx playwright test --update-snapshots
```

## 안티패턴
- Baseline 없이 매 실행마다 새 스냅샷 생성
- 동적 콘텐츠(날짜, 랜덤 이미지)가 포함된 전체 페이지 스냅샷
- 차이 임계값을 0으로 설정해 픽셀 완벽주의
- CI에서 스냅샷 업데이트 자동 승인 (리뷰 없이)

## 실전 팁
- Percy, Chromatic 같은 전용 서비스를 사용하면 변경사항을 시각적으로 리뷰할 수 있다
- 동적 요소는 `page.evaluate`로 고정값으로 대체 후 스냅샷을 찍는다
- 폰트 로딩 대기: `await page.evaluate(() => document.fonts.ready)` 후 스냅샷
- Storybook 컴포넌트에 비주얼 테스트를 연결하면 컴포넌트 단위로 관리하기 쉽다
