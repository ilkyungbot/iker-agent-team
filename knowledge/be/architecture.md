# Backend Architecture

## 핵심 원칙
- **단일 책임**: 각 서비스/모듈은 하나의 도메인만 담당
- **느슨한 결합**: 의존성 주입(DI)으로 모듈 간 결합도 최소화
- **높은 응집도**: 관련 로직을 하나의 컨텍스트 안에 모은다
- **실패 격리**: 한 서비스 장애가 전체로 전파되지 않도록 설계

## 레이어드 아키텍처 (Fastify 기준)

```
src/
├── modules/
│   └── user/
│       ├── user.controller.ts   # HTTP 요청/응답 처리
│       ├── user.service.ts      # 비즈니스 로직
│       ├── user.repository.ts   # 데이터 접근 계층
│       ├── user.schema.ts       # Zod 스키마 + Drizzle 타입
│       └── user.route.ts        # Fastify 라우트 등록
├── plugins/
│   ├── db.plugin.ts             # PostgreSQL 연결
│   ├── redis.plugin.ts          # Redis 연결
│   └── auth.plugin.ts           # JWT 데코레이터
├── shared/
│   ├── errors/
│   ├── middleware/
│   └── utils/
└── app.ts
```

## Fastify 앱 부트스트랩

```typescript
import Fastify from 'fastify'
import { dbPlugin } from './plugins/db.plugin'
import { userRoutes } from './modules/user/user.route'

export async function buildApp() {
  const app = Fastify({
    logger: { level: process.env.LOG_LEVEL ?? 'info' },
    ajv: { customOptions: { strict: 'log', keywords: ['kind', 'modifier'] } }
  })

  // 플러그인 등록 순서 중요
  await app.register(dbPlugin)
  await app.register(userRoutes, { prefix: '/api/v1/users' })

  app.setErrorHandler((error, request, reply) => {
    request.log.error(error)
    reply.status(error.statusCode ?? 500).send({
      error: error.message,
      code: error.code ?? 'INTERNAL_ERROR'
    })
  })

  return app
}
```

## 의존성 주입 패턴

```typescript
// Fastify decorators를 DI 컨테이너로 활용
declare module 'fastify' {
  interface FastifyInstance {
    userService: UserService
  }
}

// plugin 내부에서 등록
app.decorate('userService', new UserService(app.db))
```

## CQRS 경량 구현

```typescript
// Command: 상태 변경
interface CreateUserCommand {
  email: string
  name: string
}

// Query: 조회 전용 (읽기 모델 분리 가능)
interface GetUserQuery {
  userId: string
  includeDeleted?: boolean
}

class UserCommandHandler {
  async handle(cmd: CreateUserCommand): Promise<UserId> { ... }
}

class UserQueryHandler {
  async handle(query: GetUserQuery): Promise<UserDTO | null> { ... }
}
```

## 헥사고날 아키텍처 (포트/어댑터)

```typescript
// 포트 (인터페이스)
interface UserRepository {
  findById(id: string): Promise<User | null>
  save(user: User): Promise<void>
}

// 어댑터 (PostgreSQL 구현체)
class PgUserRepository implements UserRepository {
  constructor(private db: DrizzleDB) {}
  async findById(id: string) { ... }
  async save(user: User) { ... }
}

// 어댑터 (인메모리 테스트용)
class InMemoryUserRepository implements UserRepository {
  private store = new Map<string, User>()
  async findById(id: string) { return this.store.get(id) ?? null }
  async save(user: User) { this.store.set(user.id, user) }
}
```

## 체크리스트
- [ ] 레이어 간 단방향 의존성 유지 (controller → service → repository)
- [ ] 외부 서비스는 인터페이스로 추상화
- [ ] 환경 변수 검증 (zod) 앱 시작 시 수행
- [ ] Graceful shutdown 핸들러 등록
