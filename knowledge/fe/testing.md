# Testing — 프론트엔드 테스트 전략

> 테스트는 동작을 명세하는 문서다. 구현이 아닌 행동을 테스트하라.

## 핵심 원칙
- **테스트 피라미드**: Unit(70%) > Integration(20%) > E2E(10%)
- **행동 테스트**: 내부 구현이 아닌 사용자 관점의 동작 검증
- **테스트 가능한 코드**: 순수 함수, 의존성 주입으로 테스트 쉽게
- **빠른 피드백**: 단위 테스트는 100ms 이내, 전체 스위트는 30초 이내

## 도구 선택
| 용도 | 도구 |
|------|------|
| Unit / Integration | Vitest + React Testing Library |
| E2E | Playwright |
| 시각적 회귀 | Chromatic (Storybook) |
| 목 서버 | MSW (Mock Service Worker) |

## 코드 예시

❌ 구현 세부사항 테스트
```typescript
test('useState가 올바르게 호출된다', () => {
  const setState = vi.fn()
  vi.spyOn(React, 'useState').mockReturnValue([false, setState])
  // ...이건 테스트가 아니라 구현 감시
})
```

✅ 사용자 행동 테스트
```typescript
import { render, screen, userEvent } from '@testing-library/react'

test('로그인 폼: 잘못된 이메일 입력 시 에러 메시지 표시', async () => {
  const user = userEvent.setup()
  render(<LoginForm />)

  await user.type(screen.getByLabelText('이메일'), 'invalid-email')
  await user.click(screen.getByRole('button', { name: '로그인' }))

  expect(screen.getByText('올바른 이메일 형식이 아닙니다')).toBeInTheDocument()
})
```

✅ MSW로 API 목킹
```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: '김철수', email: 'kim@example.com' },
    ])
  }),
]

// 테스트에서
test('유저 목록을 로드한다', async () => {
  render(<UserList />)
  expect(await screen.findByText('김철수')).toBeInTheDocument()
})
```

✅ 커스텀 훅 테스트
```typescript
import { renderHook, act } from '@testing-library/react'

test('useCounter: increment 호출 시 count가 1 증가한다', () => {
  const { result } = renderHook(() => useCounter(0))

  act(() => result.current.increment())

  expect(result.current.count).toBe(1)
})
```

## 실전 팁
- `screen.getByRole` 우선 사용 — 접근성과 테스트 모두 강화
- `data-testid`는 마지막 수단. role, label, text로 먼저 시도
- 각 테스트는 독립적으로 실행 가능해야 함 (setup/teardown 철저히)
- Playwright: `page.getByRole`, `page.getByLabel` 사용으로 RTL과 일관성 유지

## 주의사항
- 스냅샷 테스트 남용 금지 — 변경할 때마다 업데이트하면 의미 없음
- 100% 커버리지 집착 금지 — 중요한 로직 중심으로
- 테스트가 프로덕션 코드보다 복잡하면 설계 재검토 신호
