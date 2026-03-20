# Security

## OWASP Top 10 대응

### SQL Injection → ORM 사용
```typescript
// 위험: 문자열 직접 삽입 금지
const bad = await db.execute(`SELECT * FROM users WHERE email = '${email}'`)

// 안전: Drizzle ORM 파라미터 바인딩
const safe = await db.select().from(users).where(eq(users.email, email))

// SQL 직접 사용 시 반드시 tagged template
const safe2 = await db.execute(sql`SELECT * FROM users WHERE email = ${email}`)
```

### XSS → 입력 검증 + CSP
```typescript
import DOMPurify from 'isomorphic-dompurify'

// HTML 허용 필드 sanitize
const clean = DOMPurify.sanitize(userInput, { ALLOWED_TAGS: ['b', 'i', 'em'] })

// Fastify CSP 헤더
app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:']
    }
  }
})
```

## Helmet 보안 헤더

```typescript
import helmet from '@fastify/helmet'

await app.register(helmet, {
  hsts: { maxAge: 31536000, includeSubDomains: true },
  noSniff: true,
  xssFilter: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
})
```

## 비밀번호 해싱

```typescript
import argon2 from 'argon2'

// 해싱 (회원가입/비밀번호 변경)
const hash = await argon2.hash(password, {
  type:        argon2.argon2id,
  memoryCost:  65536,  // 64MB
  timeCost:    3,
  parallelism: 4
})

// 검증 (로그인)
const isValid = await argon2.verify(storedHash, inputPassword)
```

## 민감 데이터 처리

```typescript
// 환경변수 검증
import { z } from 'zod'

const envSchema = z.object({
  DATABASE_URL:    z.string().url(),
  JWT_SECRET:      z.string().min(32),
  REDIS_URL:       z.string(),
  NODE_ENV:        z.enum(['development', 'test', 'production'])
})

export const env = envSchema.parse(process.env)

// 로그에서 민감 정보 제거
const sanitizedLog = {
  ...requestBody,
  password:    '[REDACTED]',
  creditCard:  '[REDACTED]',
  ssn:         '[REDACTED]'
}
```

## CORS 설정

```typescript
import cors from '@fastify/cors'

await app.register(cors, {
  origin: (origin, cb) => {
    const allowed = ['https://app.example.com', 'https://admin.example.com']
    if (!origin || allowed.includes(origin)) return cb(null, true)
    cb(new Error('CORS not allowed'), false)
  },
  methods:     ['GET', 'POST', 'PATCH', 'DELETE'],
  credentials: true,
  maxAge:      86400
})
```

## 입력 크기 제한

```typescript
await app.register(import('@fastify/multipart'), {
  limits: {
    fileSize:  10 * 1024 * 1024,  // 10MB
    files:     5,
    fieldSize: 1024 * 1024        // 1MB
  }
})

// Body 크기 제한
const app = Fastify({ bodyLimit: 1048576 }) // 1MB
```

## 시크릿 관리
- **절대 금지**: 시크릿을 코드/Git에 하드코딩
- **권장**: Google Secret Manager / AWS Secrets Manager
- **개발 환경**: `.env.local` (gitignore 필수)

## 체크리스트
- [ ] 모든 외부 입력 Zod로 검증
- [ ] 비밀번호 argon2id로 해싱 (bcrypt 대신)
- [ ] JWT secret 32자 이상 랜덤 문자열
- [ ] 프로덕션 DB 직접 접근 제한 (VPC/bastion)
- [ ] 의존성 취약점 정기 점검 (`npm audit`)
- [ ] Rate limiting 적용 (→ rate-limiting.md)
