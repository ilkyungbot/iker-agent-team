# 통합 테스트 — 모듈 간 경계에서 발생하는 버그를 잡는다

> 단위 테스트가 모두 통과해도 통합이 실패할 수 있다. 경계를 테스트하라.

## 핵심 원칙
- 실제 의존성(DB, 캐시, 메시지 큐)을 가능한 실제와 유사하게 구성한다
- 계약(Contract)을 검증한다: API 스펙, 이벤트 포맷, DB 스키마
- 테스트 간 DB 상태는 격리한다 (트랜잭션 롤백 또는 테스트 전용 DB)
- 느리더라도 실제 동작을 검증하는 것이 우선이다

## 판단 기준
통합 테스트 작성 대상:
- Repository 레이어 (DB 쿼리 검증)
- 외부 API 클라이언트 (실제 HTTP 호출 또는 WireMock)
- 메시지 브로커 프로듀서/컨슈머
- 인증 미들웨어 통합

격리 전략 선택:
- DB: 테스트 전용 컨테이너 (Docker) 또는 인메모리 (SQLite)
- 외부 API: MSW(Mock Service Worker) 또는 WireMock
- 메시지 큐: 인메모리 구현체

## 코드 예시 (TypeScript/Vitest)
```typescript
// src/repository/user.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest'
import { createTestDatabase, cleanupDatabase } from '../test-utils/db'
import { UserRepository } from './UserRepository'

describe('UserRepository 통합 테스트', () => {
  let db: TestDatabase
  let userRepo: UserRepository

  beforeAll(async () => {
    db = await createTestDatabase()
    userRepo = new UserRepository(db.connection)
  })

  afterAll(async () => {
    await db.close()
  })

  beforeEach(async () => {
    await cleanupDatabase(db)
  })

  it('사용자를 생성하고 ID로 조회할 수 있다', async () => {
    const created = await userRepo.create({
      email: 'test@example.com',
      name: '테스트 사용자',
    })

    const found = await userRepo.findById(created.id)
    expect(found).toMatchObject({
      email: 'test@example.com',
      name: '테스트 사용자',
    })
  })

  it('중복 이메일로 생성 시 에러를 던진다', async () => {
    await userRepo.create({ email: 'dup@example.com', name: '첫 번째' })
    await expect(
      userRepo.create({ email: 'dup@example.com', name: '두 번째' })
    ).rejects.toThrow('이메일이 이미 존재합니다')
  })
})
```

## 안티패턴
- 프로덕션 DB에 연결해서 테스트 실행
- 테스트 간 데이터 공유로 순서 의존성 발생
- 너무 많은 레이어를 한 번에 통합 테스트로 커버
- 느린 통합 테스트를 단위 테스트 스위트에 포함

## 실전 팁
- `vitest`의 `globalSetup`으로 DB 컨테이너를 한 번만 시작한다
- MSW를 활용하면 프론트엔드 통합 테스트도 쉽게 작성 가능하다
- 테스트 픽스처는 팩토리 함수로 관리한다 (`createTestUser(overrides)`)
- CI에서는 Docker Compose로 의존 서비스를 자동 시작한다
