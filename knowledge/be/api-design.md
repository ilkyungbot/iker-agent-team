# API Design

## REST 설계 원칙

### URL 네이밍
```
# 리소스는 복수 명사
GET    /api/v1/users          # 목록
POST   /api/v1/users          # 생성
GET    /api/v1/users/:id      # 단건 조회
PATCH  /api/v1/users/:id      # 부분 수정
DELETE /api/v1/users/:id      # 삭제

# 관계 표현
GET    /api/v1/users/:id/orders
POST   /api/v1/users/:id/orders

# 동사가 필요한 경우 (RPC-style 허용)
POST   /api/v1/users/:id/activate
POST   /api/v1/payments/:id/refund
```

## Fastify 라우트 정의

```typescript
import { FastifyPluginAsync } from 'fastify'
import { z } from 'zod'
import { zodToJsonSchema } from 'zod-to-json-schema'

const createUserBody = z.object({
  email: z.string().email(),
  name:  z.string().min(1).max(100),
  role:  z.enum(['admin', 'member']).default('member')
})

const userParams = z.object({
  id: z.string().uuid()
})

export const userRoutes: FastifyPluginAsync = async (app) => {
  app.post('/', {
    schema: {
      body: zodToJsonSchema(createUserBody),
      response: { 201: zodToJsonSchema(userResponseSchema) }
    },
    handler: async (req, reply) => {
      const body = createUserBody.parse(req.body)
      const user = await app.userService.create(body)
      reply.status(201).send(user)
    }
  })

  app.get('/:id', {
    schema: { params: zodToJsonSchema(userParams) },
    handler: async (req, reply) => {
      const { id } = userParams.parse(req.params)
      const user = await app.userService.findById(id)
      if (!user) return reply.status(404).send({ code: 'USER_NOT_FOUND' })
      return user
    }
  })
}
```

## 표준 응답 포맷

```typescript
// 성공 응답
interface SuccessResponse<T> {
  data: T
  meta?: {
    total?: number
    page?: number
    limit?: number
    cursor?: string
  }
}

// 에러 응답
interface ErrorResponse {
  code:    string        // 'USER_NOT_FOUND', 'VALIDATION_ERROR'
  message: string        // 사람이 읽을 수 있는 설명
  details?: unknown      // 검증 오류 상세
  requestId: string      // 추적용
}
```

## 페이지네이션

```typescript
// Cursor 기반 (대용량 데이터 권장)
const cursorPaginationQuery = z.object({
  cursor: z.string().optional(),
  limit:  z.coerce.number().int().min(1).max(100).default(20),
  order:  z.enum(['asc', 'desc']).default('desc')
})

async function paginateUsers(query: z.infer<typeof cursorPaginationQuery>) {
  const { cursor, limit, order } = query
  const rows = await db.select()
    .from(users)
    .where(cursor ? lt(users.id, cursor) : undefined)
    .orderBy(order === 'desc' ? desc(users.createdAt) : asc(users.createdAt))
    .limit(limit + 1)  // 다음 페이지 존재 여부 확인

  const hasNext = rows.length > limit
  return {
    data: rows.slice(0, limit),
    meta: { hasNext, nextCursor: hasNext ? rows[limit - 1].id : null }
  }
}
```

## API 버저닝 전략

```typescript
// URL 버저닝 (권장)
app.register(v1Routes, { prefix: '/api/v1' })
app.register(v2Routes, { prefix: '/api/v2' })

// 헤더 버저닝 (선택적)
app.addHook('onRequest', async (req) => {
  const version = req.headers['api-version'] ?? 'v1'
  req.apiVersion = version
})
```

## 응답 직렬화 최적화

```typescript
// fast-json-stringify로 자동 최적화 (Fastify 기본)
// schema 정의 시 30-40% 빠른 직렬화
const responseSchema = {
  type: 'object',
  properties: {
    id:    { type: 'string' },
    email: { type: 'string' },
    name:  { type: 'string' }
  }
}
```

## 체크리스트
- [ ] 모든 엔드포인트에 JSON schema 정의
- [ ] 에러 응답에 requestId 포함
- [ ] 페이지네이션 있는 목록 API는 cursor 방식 사용
- [ ] HATEOAS 불필요 — 단순한 구조 유지
