# Testing

## 테스트 전략 (테스트 피라미드)

```
        /\
       /E2E\       소수 (핵심 플로우만)
      /------\
     /통합 테스트\   중간 (API 엔드포인트)
    /------------\
   /   단위 테스트  \  다수 (서비스/유틸리티)
  /-----------------\
```

## 설정 (Vitest)

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals:     true,
    environment: 'node',
    coverage: {
      provider:    'v8',
      reporter:    ['text', 'lcov', 'html'],
      thresholds:  { lines: 80, functions: 80, branches: 70 }
    },
    setupFiles: ['./test/setup.ts']
  }
})
```

## 단위 테스트

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { UserService } from './user.service'

describe('UserService', () => {
  let service: UserService
  let mockRepo: MockUserRepository

  beforeEach(() => {
    mockRepo = new InMemoryUserRepository()
    service = new UserService(mockRepo)
  })

  it('이메일 중복 시 ConflictError를 던진다', async () => {
    await mockRepo.save(buildUser({ email: 'test@test.com' }))

    await expect(
      service.create({ email: 'test@test.com', name: 'Test' })
    ).rejects.toThrow(ConflictError)
  })

  it('사용자를 정상 생성한다', async () => {
    const user = await service.create({ email: 'new@test.com', name: 'New' })
    expect(user).toMatchObject({ email: 'new@test.com', name: 'New' })
    expect(user.id).toBeDefined()
  })
})
```

## API 통합 테스트

```typescript
import { buildApp } from '../src/app'
import { buildTestDb, cleanupDb } from './helpers/db'

describe('POST /api/v1/users', () => {
  let app: FastifyInstance

  beforeAll(async () => {
    app = await buildApp({ db: buildTestDb() })
    await app.ready()
  })

  afterAll(async () => {
    await cleanupDb()
    await app.close()
  })

  it('201 + 생성된 사용자 반환', async () => {
    const response = await app.inject({
      method: 'POST',
      url:    '/api/v1/users',
      payload: { email: 'test@test.com', name: 'Test User' }
    })

    expect(response.statusCode).toBe(201)
    const body = response.json()
    expect(body).toMatchObject({ email: 'test@test.com' })
    expect(body.id).toMatch(/^[0-9a-f-]{36}$/)
  })

  it('이메일 없으면 400 반환', async () => {
    const response = await app.inject({
      method:  'POST',
      url:     '/api/v1/users',
      payload: { name: 'No Email' }
    })
    expect(response.statusCode).toBe(400)
    expect(response.json().code).toBe('VALIDATION_ERROR')
  })
})
```

## 테스트 픽스처 (Factory 패턴)

```typescript
import { faker } from '@faker-js/faker'

export function buildUser(overrides: Partial<User> = {}): User {
  return {
    id:        faker.string.uuid(),
    email:     faker.internet.email(),
    name:      faker.person.fullName(),
    role:      'member',
    isActive:  true,
    createdAt: new Date(),
    updatedAt: new Date(),
    deletedAt: null,
    ...overrides
  }
}

export function buildOrder(overrides: Partial<Order> = {}): Order {
  return {
    id:        faker.string.uuid(),
    userId:    faker.string.uuid(),
    status:    'pending',
    total:     faker.number.int({ min: 1000, max: 100000 }),
    createdAt: new Date(),
    ...overrides
  }
}
```

## 체크리스트
- [ ] 서비스 레이어 단위 테스트 (Mock DB)
- [ ] API 엔드포인트 통합 테스트 (실제 DB 사용)
- [ ] 에러 케이스 테스트 포함
- [ ] CI에서 커버리지 80% 이상 강제
- [ ] 테스트 DB는 프로덕션 DB와 분리
