# Authentication

## JWT 인증 플로우

```
클라이언트 → POST /auth/login → 서버
서버 → Access Token (15m) + Refresh Token (7d) → 클라이언트
클라이언트 → Authorization: Bearer <access_token> → 서버
Access Token 만료 → POST /auth/refresh → 새 Access Token
```

## JWT 구현

```typescript
import jwt from 'jsonwebtoken'
import { z } from 'zod'

const tokenPayloadSchema = z.object({
  sub:   z.string().uuid(),
  email: z.string().email(),
  role:  z.enum(['admin', 'member']),
  iat:   z.number(),
  exp:   z.number()
})

type TokenPayload = z.infer<typeof tokenPayloadSchema>

const JWT_SECRET          = process.env.JWT_SECRET!
const ACCESS_TOKEN_TTL    = '15m'
const REFRESH_TOKEN_TTL   = '7d'

function signAccessToken(payload: Omit<TokenPayload, 'iat' | 'exp'>): string {
  return jwt.sign(payload, JWT_SECRET, { expiresIn: ACCESS_TOKEN_TTL, algorithm: 'HS256' })
}

function verifyToken(token: string): TokenPayload {
  const decoded = jwt.verify(token, JWT_SECRET)
  return tokenPayloadSchema.parse(decoded)
}
```

## Fastify 인증 플러그인

```typescript
import fp from 'fastify-plugin'
import { FastifyRequest } from 'fastify'

declare module 'fastify' {
  interface FastifyRequest {
    user: TokenPayload
  }
}

export const authPlugin = fp(async (app) => {
  app.decorateRequest('user', null)

  app.addHook('onRequest', async (req, reply) => {
    const skipPaths = ['/health', '/metrics', '/api/v1/auth/login', '/api/v1/auth/refresh']
    if (skipPaths.some(p => req.url.startsWith(p))) return

    const authHeader = req.headers.authorization
    if (!authHeader?.startsWith('Bearer ')) {
      return reply.status(401).send({ code: 'MISSING_TOKEN' })
    }

    try {
      const token = authHeader.slice(7)
      req.user = verifyToken(token)
    } catch (err) {
      if (err instanceof jwt.TokenExpiredError) {
        return reply.status(401).send({ code: 'TOKEN_EXPIRED' })
      }
      return reply.status(401).send({ code: 'INVALID_TOKEN' })
    }
  })
})
```

## Refresh Token 로직

```typescript
// Refresh Token은 DB에 저장 (무효화 가능하게)
const refreshTokens = pgTable('refresh_tokens', {
  id:        uuid('id').primaryKey().defaultRandom(),
  userId:    uuid('user_id').notNull().references(() => users.id),
  token:     text('token').notNull().unique(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  revokedAt: timestamp('revoked_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow()
})

async function refreshAccessToken(refreshToken: string) {
  const stored = await db.select()
    .from(refreshTokens)
    .where(and(
      eq(refreshTokens.token, refreshToken),
      isNull(refreshTokens.revokedAt),
      gt(refreshTokens.expiresAt, new Date())
    ))
    .limit(1)

  if (!stored[0]) throw new UnauthorizedError('Invalid refresh token')

  const user = await getUserById(stored[0].userId)
  return signAccessToken({ sub: user.id, email: user.email, role: user.role })
}
```

## 소셜 로그인 (OAuth2)

```typescript
import { OAuth2Client } from 'google-auth-library'

const googleClient = new OAuth2Client(process.env.GOOGLE_CLIENT_ID)

async function verifyGoogleToken(idToken: string) {
  const ticket = await googleClient.verifyIdToken({
    idToken,
    audience: process.env.GOOGLE_CLIENT_ID
  })
  const payload = ticket.getPayload()
  if (!payload?.email) throw new UnauthorizedError('Invalid Google token')
  return { email: payload.email, name: payload.name ?? '', avatar: payload.picture }
}
```

## 체크리스트
- [ ] Access Token 짧게 (15분 이하)
- [ ] Refresh Token DB 저장 (무효화 가능)
- [ ] JWT secret 32자 이상 랜덤
- [ ] 로그아웃 시 Refresh Token 즉시 무효화
- [ ] 비밀번호 argon2id 해싱 (→ security.md)
