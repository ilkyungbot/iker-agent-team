# 모킹 — 테스트를 빠르고 예측 가능하게 만드는 기술

> 모킹은 격리를 위한 도구다. 남용하면 테스트가 구현과 결합된다.

## 핵심 원칙
- 외부 시스템(API, DB, 이메일)만 모킹한다
- 비즈니스 로직은 모킹하지 않는다
- Mock보다 Fake(실제 구현체 경량 버전)를 선호한다
- 모킹은 계약(인터페이스)을 기준으로 한다

## 판단 기준
| 방법 | 사용 시점 |
|------|-----------|
| Stub | 정해진 값을 반환해야 할 때 |
| Mock | 호출 여부/횟수를 검증해야 할 때 |
| Spy | 실제 구현을 유지하면서 호출을 추적할 때 |
| Fake | 빠른 인메모리 대체 구현이 필요할 때 |
| MSW | HTTP 요청을 네트워크 레벨에서 가로챌 때 |

## 코드 예시 (TypeScript/Vitest)
```typescript
// vi.mock으로 모듈 모킹
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { sendWelcomeEmail } from '../email/emailService'
import { registerUser } from './userService'

vi.mock('../email/emailService', () => ({
  sendWelcomeEmail: vi.fn().mockResolvedValue({ success: true }),
}))

describe('사용자 등록 서비스', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('등록 후 환영 이메일을 발송한다', async () => {
    await registerUser({ email: 'new@test.com', password: 'secure123' })

    expect(sendWelcomeEmail).toHaveBeenCalledOnce()
    expect(sendWelcomeEmail).toHaveBeenCalledWith('new@test.com')
  })
})

// MSW로 HTTP 모킹
import { setupServer } from 'msw/node'
import { http, HttpResponse } from 'msw'

const server = setupServer(
  http.get('https://api.external.com/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: '테스트 사용자',
    })
  }),
  http.post('https://payment.example.com/charge', () => {
    return HttpResponse.json({ status: 'success', transactionId: 'tx_123' })
  })
)

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

## 안티패턴
- 테스트하려는 코드 자체를 모킹 (의미 없는 테스트)
- `vi.mock`을 남용해 실제 통합 문제를 숨김
- 모킹 후 `expect(mock).toHaveBeenCalled()` 없이 테스트 통과
- 타입 없는 모킹 (`as any` 남발)

## 실전 팁
- `vi.spyOn`은 실제 구현을 유지하므로 더 안전하다
- Fake Repository 패턴으로 DB 모킹을 인터페이스 기반으로 관리한다
- `server.use(http.get(...))` 으로 특정 테스트에서 에러 응답을 오버라이드한다
- 모킹 초기화는 `beforeEach`에서 `vi.clearAllMocks()`로 일괄 처리한다
