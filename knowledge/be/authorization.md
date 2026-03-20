# Authorization

## RBAC (Role-Based Access Control)

```typescript
// 역할 정의
type Role       = 'admin' | 'manager' | 'member' | 'guest'
type Permission = 'users:read' | 'users:write' | 'users:delete'
                | 'orders:read' | 'orders:write' | 'orders:cancel'
                | 'reports:read' | 'settings:write'

const rolePermissions: Record<Role, Permission[]> = {
  admin:   ['users:read', 'users:write', 'users:delete', 'orders:read', 'orders:write', 'orders:cancel', 'reports:read', 'settings:write'],
  manager: ['users:read', 'orders:read', 'orders:write', 'orders:cancel', 'reports:read'],
  member:  ['orders:read', 'orders:write'],
  guest:   ['orders:read']
}

function hasPermission(role: Role, permission: Permission): boolean {
  return rolePermissions[role]?.includes(permission) ?? false
}
```

## Fastify 권한 데코레이터

```typescript
import fp from 'fastify-plugin'

declare module 'fastify' {
  interface FastifyInstance {
    requirePermission: (permission: Permission) => preHandlerHookHandler
    requireRole:       (roles: Role[])          => preHandlerHookHandler
  }
}

export const authorizationPlugin = fp(async (app) => {
  app.decorate('requirePermission', (permission: Permission) => {
    return async (req: FastifyRequest, reply: FastifyReply) => {
      if (!req.user) return reply.status(401).send({ code: 'UNAUTHORIZED' })

      if (!hasPermission(req.user.role, permission)) {
        return reply.status(403).send({
          code:    'FORBIDDEN',
          message: `Permission '${permission}' required`
        })
      }
    }
  })

  app.decorate('requireRole', (roles: Role[]) => {
    return async (req: FastifyRequest, reply: FastifyReply) => {
      if (!req.user) return reply.status(401).send({ code: 'UNAUTHORIZED' })
      if (!roles.includes(req.user.role)) {
        return reply.status(403).send({ code: 'FORBIDDEN' })
      }
    }
  })
})

// 사용
app.delete('/api/v1/users/:id', {
  preHandler: [app.requirePermission('users:delete')],
  handler: deleteUserHandler
})
```

## 리소스 기반 접근 제어 (ABAC 경량)

```typescript
// 본인 리소스만 접근 가능 패턴
async function getUserOrder(req: AuthenticatedRequest) {
  const order = await orderService.findById(req.params.id)
  if (!order) throw new NotFoundError('Order', req.params.id)

  // 본인 주문 또는 어드민만 접근 가능
  if (order.userId !== req.user.sub && req.user.role !== 'admin') {
    throw new ForbiddenError('Access denied')
  }

  return order
}
```

## Policy 패턴 (확장 가능한 권한)

```typescript
interface Policy<T> {
  canRead(user: TokenPayload, resource: T): boolean
  canUpdate(user: TokenPayload, resource: T): boolean
  canDelete(user: TokenPayload, resource: T): boolean
}

class OrderPolicy implements Policy<Order> {
  canRead(user: TokenPayload, order: Order): boolean {
    return user.role === 'admin' || order.userId === user.sub
  }

  canUpdate(user: TokenPayload, order: Order): boolean {
    return (user.role === 'admin' || order.userId === user.sub)
      && order.status === 'pending'
  }

  canDelete(user: TokenPayload, order: Order): boolean {
    return user.role === 'admin'
  }
}

const orderPolicy = new OrderPolicy()

// 사용
if (!orderPolicy.canUpdate(req.user, order)) throw new ForbiddenError()
```

## API Key 인증 (서버 간 통신)

```typescript
const apiKeys = pgTable('api_keys', {
  id:         uuid('id').primaryKey().defaultRandom(),
  keyHash:    text('key_hash').notNull().unique(),  // SHA-256 해시 저장
  name:       text('name').notNull(),
  scopes:     text('scopes').array().notNull(),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  expiresAt:  timestamp('expires_at', { withTimezone: true })
})

async function verifyApiKey(rawKey: string) {
  const hash = crypto.createHash('sha256').update(rawKey).digest('hex')
  const key  = await db.select().from(apiKeys).where(eq(apiKeys.keyHash, hash)).limit(1)
  if (!key[0] || (key[0].expiresAt && key[0].expiresAt < new Date())) {
    throw new UnauthorizedError('Invalid API key')
  }
  return key[0]
}
```

## 체크리스트
- [ ] 역할-권한 매핑 중앙 관리
- [ ] 리소스 소유권 검증 (본인 리소스)
- [ ] 권한 없음 vs 인증 없음 구분 (403 vs 401)
- [ ] API Key는 원문 저장 금지 (해시만 저장)
- [ ] 권한 변경 시 기존 토큰 무효화 방안
