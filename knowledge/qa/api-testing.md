# API 테스트 — HTTP 계약을 코드로 검증한다

> API는 팀 간 계약이다. 계약이 깨지면 클라이언트가 조용히 망가진다.

## 핵심 원칙
- 응답 스키마, 상태 코드, 헤더를 모두 검증한다
- Happy Path와 에러 케이스를 동등하게 다룬다
- 인증/인가 경계를 반드시 테스트한다
- 계약 테스트(Contract Testing)로 프로바이더-컨슈머 불일치를 조기 발견한다

## 판단 기준
API 테스트 필수 케이스:
- 정상 요청 → 올바른 응답 스키마
- 인증 없는 요청 → 401
- 권한 없는 요청 → 403
- 잘못된 입력 → 400 (에러 메시지 포함)
- 존재하지 않는 리소스 → 404
- 중복 생성 → 409

## 코드 예시 (TypeScript/Vitest)
```typescript
// src/api/users.api.test.ts
import { describe, it, expect, beforeAll } from 'vitest'
import { createApp } from '../app'
import { createTestToken } from '../test-utils/auth'

describe('POST /api/users', () => {
  let app: App
  let adminToken: string

  beforeAll(async () => {
    app = await createApp({ env: 'test' })
    adminToken = createTestToken({ role: 'admin' })
  })

  it('유효한 데이터로 사용자를 생성한다', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      headers: { Authorization: `Bearer ${adminToken}` },
      payload: { email: 'new@test.com', name: '신규 사용자', role: 'user' },
    })

    expect(response.statusCode).toBe(201)
    const body = response.json()
    expect(body).toMatchObject({
      id: expect.any(String),
      email: 'new@test.com',
      createdAt: expect.any(String),
    })
    // 민감 정보 노출 확인
    expect(body).not.toHaveProperty('password')
    expect(body).not.toHaveProperty('passwordHash')
  })

  it('인증 토큰 없이 요청 시 401을 반환한다', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      payload: { email: 'test@test.com', name: '테스트' },
    })
    expect(response.statusCode).toBe(401)
  })

  it('이메일 형식이 잘못된 경우 400을 반환한다', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      headers: { Authorization: `Bearer ${adminToken}` },
      payload: { email: 'invalid-email', name: '테스트' },
    })
    expect(response.statusCode).toBe(400)
    expect(response.json().message).toContain('이메일')
  })
})
```

## 안티패턴
- 응답 상태 코드만 검증하고 본문 스키마는 미검증
- 테스트용 실제 이메일 발송, 결제 처리
- 환경 변수에 프로덕션 API 키 사용
- 병렬 실행 시 동일 리소스 ID로 충돌

## 실전 팁
- `supertest` 또는 프레임워크 내장 `inject`를 활용해 서버를 직접 띄운다
- Zod 스키마로 응답 타입을 검증하면 타입 안전성도 확보된다
- 페이지네이션 테스트는 `limit=2` 같은 작은 값으로 경계를 검증한다
- 레이트 리밋 테스트는 별도 slow 태그로 분리해 CI에서 선택적으로 실행한다
