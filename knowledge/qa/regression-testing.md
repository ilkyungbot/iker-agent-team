# 회귀 테스트 — 고친 버그가 다시 나타나지 않도록 한다

> 회귀는 예방 가능하다. 버그가 발생했다면 테스트가 없었기 때문이다.

## 핵심 원칙
- 버그 수정 전 반드시 해당 버그를 재현하는 테스트를 작성한다
- 회귀 테스트는 영구적으로 유지한다 (삭제 금지)
- 배포 전 회귀 스위트를 반드시 실행한다
- 버그 ID를 테스트에 주석으로 남긴다

## 판단 기준
회귀 테스트 트리거:
- 모든 PR 머지 전
- 릴리즈 브랜치 생성 시
- 핫픽스 배포 전
- 주요 리팩터링 완료 후

회귀 위험도가 높은 영역:
- 공유 유틸리티 함수 변경
- DB 스키마 마이그레이션
- 외부 API 클라이언트 업데이트
- 인증 미들웨어 수정

## 코드 예시 (TypeScript/Vitest)
```typescript
// src/billing/invoice.regression.test.ts
/**
 * 회귀 테스트 모음
 * 각 테스트는 프로덕션에서 발생한 버그에 대응한다
 */

describe('청구서 생성 회귀 테스트', () => {
  // BUG-234: 연간 구독을 월간으로 다운그레이드 시 청구서 금액이 0원이 됨
  // 수정일: 2026-02-15, 수정자: @devteam
  it('[BUG-234] 다운그레이드 시 일할 계산이 올바르게 적용된다', () => {
    const subscription = createTestSubscription({
      plan: 'yearly',
      startDate: new Date('2026-01-01'),
    })

    const invoice = generateDowngradeInvoice(subscription, {
      newPlan: 'monthly',
      downgradeDate: new Date('2026-01-15'),
    })

    // 15일 사용분 환불 + 새 플랜 청구
    expect(invoice.creditAmount).toBeGreaterThan(0)
    expect(invoice.chargeAmount).toBeGreaterThan(0)
    expect(invoice.totalAmount).not.toBe(0)
  })

  // BUG-289: 쿠폰 코드가 대소문자를 구분해서 같은 쿠폰이 두 번 적용됨
  it('[BUG-289] 쿠폰 코드는 대소문자를 구분하지 않는다', () => {
    const result1 = applyCoupon('WELCOME10')
    const result2 = applyCoupon('welcome10')
    const result3 = applyCoupon('Welcome10')

    expect(result1.discountRate).toBe(result2.discountRate)
    expect(result2.discountRate).toBe(result3.discountRate)
  })
})
```

## 안티패턴
- 버그 수정 후 테스트 작성을 건너뜀
- 회귀 테스트를 별도 폴더에 격리해 관리 소홀
- 자동화보다 수동 체크리스트에 의존
- 버그 ID 없이 테스트만 추가해 맥락 상실

## 실전 팁
- 버그 수정 PR에 회귀 테스트를 의무적으로 포함한다 (PR 템플릿에 체크리스트 추가)
- 회귀 스위트는 `describe('회귀 테스트', () => {})` 블록으로 그룹화한다
- 플레이키 회귀 테스트는 즉시 조사한다 (인프라 문제 vs 코드 문제)
- 분기별 회귀 테스트 커버리지를 리뷰하고 누락된 버그 케이스를 추가한다
