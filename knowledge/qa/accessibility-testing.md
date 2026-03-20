# 접근성 테스트 — 모든 사용자가 사용할 수 있어야 한다

> 접근성은 특수 기능이 아니라 기본 품질 요건이다. 스크린 리더 사용자도 고객이다.

## 핵심 원칙
- WCAG 2.1 AA 기준을 최소 준수 목표로 설정한다
- 자동화 도구는 접근성 문제의 30-40%만 탐지한다. 수동 테스트가 필수다
- 키보드만으로 모든 기능을 사용할 수 있어야 한다
- 색상만으로 정보를 전달하지 않는다

## 판단 기준
자동화 가능 항목:
- 이미지 `alt` 텍스트 누락
- 폼 레이블 연결 미흡
- 색상 대비 비율 (4.5:1 이상)
- ARIA 속성 오용
- 포커스 순서 오류

수동 테스트 필수 항목:
- 스크린 리더(VoiceOver/NVDA) 탐색
- 키보드 탭 순서의 논리적 흐름
- 모달/드롭다운에서 포커스 트랩
- 동적 콘텐츠 변경 시 스크린 리더 알림

## 코드 예시 (Playwright + axe-core)
```typescript
// e2e/accessibility.spec.ts
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test.describe('접근성 자동화 검사', () => {
  test('홈페이지는 WCAG AA 기준을 위반하지 않는다', async ({ page }) => {
    await page.goto('/')

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
      .analyze()

    expect(results.violations).toEqual([])
  })

  test('결제 폼은 키보드만으로 완료할 수 있다', async ({ page }) => {
    await page.goto('/checkout')

    // Tab으로만 이동하며 폼 완료
    await page.keyboard.press('Tab')
    await expect(page.getByLabel('카드 번호')).toBeFocused()
    await page.keyboard.type('4242424242424242')

    await page.keyboard.press('Tab')
    await expect(page.getByLabel('만료일')).toBeFocused()
  })

  test('에러 메시지가 스크린 리더에 전달된다', async ({ page }) => {
    await page.goto('/login')
    await page.getByRole('button', { name: '로그인' }).click()

    // aria-live 영역에 에러가 표시되는지 확인
    await expect(page.locator('[role="alert"]')).toBeVisible()
    await expect(page.locator('[aria-live="polite"]')).toContainText('필수 항목')
  })
})
```

## 안티패턴
- `div`와 `span`으로만 인터랙티브 요소 구현
- `tabindex="-1"` 없이 포커스 관리
- 색상 대비를 "디자인 의도"라며 무시
- `aria-hidden="true"`를 접근성 문제 해결에 남용

## 실전 팁
- Chrome 확장 프로그램 `axe DevTools`로 개발 중 즉시 확인한다
- Lighthouse의 접근성 점수를 CI에서 95점 이상 강제한다
- 컴포넌트 라이브러리 선택 시 접근성 지원 여부를 필수 기준으로 삼는다
- 분기별 스크린 리더 수동 테스트를 팀 일정에 포함한다
