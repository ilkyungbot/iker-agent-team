# 테스트 데이터 — 예측 가능하고 재사용 가능한 데이터 관리

> 테스트 데이터가 오염되면 테스트 결과를 신뢰할 수 없다.

## 핵심 원칙
- 각 테스트는 필요한 데이터를 직접 생성하고 정리한다
- 픽스처(fixture) 팩토리로 기본값과 오버라이드를 관리한다
- 프로덕션 데이터를 테스트에 직접 사용하지 않는다
- 민감 데이터는 마스킹 후 사용한다

## 판단 기준
데이터 관리 전략:
| 전략 | 사용 시점 |
|------|-----------|
| 인라인 데이터 | 단순 단위 테스트 |
| 팩토리 함수 | 재사용되는 엔티티 생성 |
| 시드 데이터 | 읽기 전용 참조 데이터 |
| 스냅샷 | 복잡한 객체 구조 비교 |

격리 전략:
- 단위 테스트: 인라인, 모킹
- 통합 테스트: 트랜잭션 롤백 또는 테스트 DB 초기화
- E2E: API로 생성/삭제, 테스트별 유니크 ID

## 코드 예시 (TypeScript/Vitest)
```typescript
// src/test-utils/factories.ts — 팩토리 패턴
import { faker } from '@faker-js/faker'

interface UserOverrides {
  id?: string
  email?: string
  name?: string
  role?: 'user' | 'admin'
  plan?: 'free' | 'pro' | 'enterprise'
}

export function createTestUser(overrides: UserOverrides = {}): User {
  return {
    id: overrides.id ?? faker.string.uuid(),
    email: overrides.email ?? faker.internet.email(),
    name: overrides.name ?? faker.person.fullName(),
    role: overrides.role ?? 'user',
    plan: overrides.plan ?? 'free',
    createdAt: new Date(),
  }
}

export function createTestSubscription(overrides = {}) {
  return {
    id: faker.string.uuid(),
    userId: faker.string.uuid(),
    planId: 'pro',
    status: 'active',
    startDate: faker.date.past(),
    nextBillingDate: faker.date.future(),
    ...overrides,
  }
}

// 사용 예시
it('프로 플랜 사용자는 고급 기능에 접근할 수 있다', () => {
  const proUser = createTestUser({ plan: 'pro' })
  expect(canAccessAdvancedFeature(proUser)).toBe(true)
})

it('무료 플랜 사용자는 고급 기능에 접근할 수 없다', () => {
  const freeUser = createTestUser({ plan: 'free' })
  expect(canAccessAdvancedFeature(freeUser)).toBe(false)
})
```

## 안티패턴
- `testUser1`, `testUser2` 같은 정적 하드코딩 데이터
- 테스트 간 데이터 공유로 순서 의존성 발생
- 실제 이메일/전화번호를 테스트 데이터로 사용
- 테스트 후 데이터 정리 누락으로 DB 오염

## 실전 팁
- `@faker-js/faker`로 현실적인 무작위 데이터를 생성한다
- 씨드 값을 고정하면 재현 가능한 무작위 데이터를 만들 수 있다 (`faker.seed(42)`)
- 복잡한 계층 구조 데이터는 Builder 패턴으로 관리한다
- 데이터베이스 시드 스크립트는 `npm run db:seed:test`로 분리한다
