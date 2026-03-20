# E2E 테스트 — 사용자 여정을 끝에서 끝까지 검증한다

> E2E는 느리고 비싸다. 그래서 Critical Path만 커버해야 한다.

## 핵심 원칙
- 실제 브라우저에서 실제 사용자처럼 동작한다
- 테스트 시나리오는 비즈니스 가치 기준으로 선정한다
- 전체 E2E 스위트는 20분 이내에 완료되어야 한다
- Flaky test는 팀 신뢰도를 파괴한다. 즉시 수정하거나 격리한다

## 판단 기준
E2E 테스트 필수 대상 (Critical Path):
- 회원가입 → 로그인 → 핵심 기능 사용 → 로그아웃
- 결제 완료 플로우
- 데이터 생성 → 수정 → 삭제 사이클
- 권한별 접근 제어

E2E 테스트 제외 대상:
- 단순 UI 레이아웃 확인 (Visual 테스트로 대체)
- 유틸 함수 로직 (단위 테스트로 대체)
- 에러 메시지 텍스트 (단위 테스트로 대체)

## 코드 예시 (Playwright)
```typescript
// e2e/subscription-flow.spec.ts
import { test, expect } from '@playwright/test'
import { loginAs, createTestUser } from './helpers'

test.describe('구독 신청 플로우', () => {
  test.beforeEach(async ({ page }) => {
    const user = await createTestUser()
    await loginAs(page, user)
  })

  test('무료 플랜에서 프로 플랜으로 업그레이드', async ({ page }) => {
    await page.goto('/dashboard')
    await page.getByRole('button', { name: '플랜 업그레이드' }).click()

    await expect(page.getByText('프로 플랜')).toBeVisible()
    await page.getByRole('button', { name: '프로 플랜 선택' }).click()

    // 결제 정보 입력
    await page.getByLabel('카드 번호').fill('4242424242424242')
    await page.getByLabel('만료일').fill('12/28')
    await page.getByLabel('CVC').fill('123')
    await page.getByRole('button', { name: '결제 완료' }).click()

    await expect(page.getByText('프로 플랜이 활성화되었습니다')).toBeVisible()
  })
})
```

## 안티패턴
- E2E로 모든 테스트 케이스를 커버하려는 시도
- `sleep`/`waitForTimeout`으로 타이밍 문제 해결 (명시적 대기 사용)
- 프로덕션 DB/API에 연결해서 E2E 실행
- 테스트 실패 시 원인 파악 없이 재실행만 반복

## 실전 팁
- `test.step`으로 긴 시나리오를 의미 있는 단계로 분리한다
- `page.screenshot({ path: 'failure.png' })`로 실패 시 스크린샷을 자동 저장한다
- 테스트 계정과 데이터는 테스트 전 API로 생성하고 후에 정리한다
- 병렬 실행 시 데이터 충돌을 막기 위해 테스트별 유니크 식별자를 사용한다
