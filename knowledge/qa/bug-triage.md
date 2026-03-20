# 버그 트리아지 — 모든 버그가 동등하지 않다

> 버그를 빨리 고치는 것보다 중요한 버그를 먼저 고치는 것이 중요하다.

## 핵심 원칙
- 심각도(Severity)와 우선순위(Priority)는 다르다
- 트리아지는 24시간 이내에 완료한다
- 재현 불가 버그는 최소 3번 재현 시도 후 보류 처리
- 버그 수정 전 반드시 실패하는 테스트를 먼저 작성한다

## 판단 기준
**심각도 분류:**
| 등급 | 정의 | SLA |
|------|------|-----|
| Critical (P0) | 서비스 전체 다운, 데이터 손실 | 즉시 대응 |
| High (P1) | 핵심 기능 불가, 보안 취약점 | 24시간 이내 |
| Medium (P2) | 기능 동작하나 오동작, 우회 가능 | 1주일 이내 |
| Low (P3) | UI 이슈, 불편하지만 동작 | 다음 스프린트 |

**트리아지 체크리스트:**
- [ ] 재현 가능한가? (재현 단계 명시)
- [ ] 영향 범위는? (전체/일부 사용자)
- [ ] 언제부터 발생했나? (회귀 버그?)
- [ ] 임시 해결책이 있는가?
- [ ] 담당자 지정

## 코드 예시 (버그 수정 프로세스)
```typescript
// Step 1: 실패하는 테스트 먼저 작성 (버그 재현)
it('[BUG-123] 구독 취소 후 재구독 시 할인이 중복 적용된다', () => {
  // 재현 조건
  const user = createUserWithCancelledSubscription()
  const resubscribeResult = resubscribe(user, { couponCode: 'WELCOME10' })

  // 기대값: 10% 할인 1회만 적용
  expect(resubscribeResult.discountAmount).toBe(1000)  // FAIL
  // 실제: 20% 할인이 적용됨 (버그)
})

// Step 2: 수정 후 테스트 통과 확인
function resubscribe(user: User, options: ResubscribeOptions) {
  // 기존 할인 쿠폰 초기화 후 새 쿠폰 적용
  const baseSubscription = resetDiscounts(user.subscription)
  return applyDiscount(baseSubscription, options.couponCode)
}

// Step 3: 회귀 방지를 위해 테스트 영구 유지
// (테스트 파일에 BUG-123 레퍼런스 유지)
```

## 안티패턴
- 버그 수정 후 테스트 없이 배포
- Critical 버그를 "다음 스프린트에" 미루기
- 재현 단계 없이 버그 등록 ("가끔 안 됨")
- 원인 파악 없이 증상만 수정

## 실전 팁
- 버그 리포트 템플릿: 재현 단계 / 실제 결과 / 기대 결과 / 환경 정보
- Sentry/DataDog에서 에러 빈도와 영향 사용자 수를 우선순위 판단에 활용한다
- 같은 컴포넌트에서 버그가 반복된다면 리팩터링 신호다
- P0 핫픽스는 별도 브랜치에서 빠르게 배포하고 main에 백머지한다
