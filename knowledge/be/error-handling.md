# Error Handling

## 커스텀 에러 클래스

```typescript
export class AppError extends Error {
  constructor(
    message: string,
    public readonly code:       string,
    public readonly statusCode: number = 500,
    public readonly details?:   unknown
  ) {
    super(message)
    this.name = 'AppError'
    Error.captureStackTrace(this, this.constructor)
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} not found: ${id}`, 'NOT_FOUND', 404)
  }
}

export class ValidationError extends AppError {
  constructor(details: unknown) {
    super('Validation failed', 'VALIDATION_ERROR', 400, details)
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 'CONFLICT', 409)
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 'UNAUTHORIZED', 401)
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 'FORBIDDEN', 403)
  }
}
```

## Fastify 전역 에러 핸들러

```typescript
import { FastifyError } from 'fastify'
import { ZodError } from 'zod'

app.setErrorHandler((error, request, reply) => {
  const requestId = request.id

  // Zod 검증 에러
  if (error instanceof ZodError) {
    return reply.status(400).send({
      code:      'VALIDATION_ERROR',
      message:   'Invalid request data',
      details:   error.flatten(),
      requestId
    })
  }

  // 앱 정의 에러
  if (error instanceof AppError) {
    request.log.warn({ err: error, requestId }, error.message)
    return reply.status(error.statusCode).send({
      code:      error.code,
      message:   error.message,
      details:   error.details,
      requestId
    })
  }

  // Fastify 에러 (400 범위)
  if (error.statusCode && error.statusCode < 500) {
    return reply.status(error.statusCode).send({
      code:      'REQUEST_ERROR',
      message:   error.message,
      requestId
    })
  }

  // 예상치 못한 에러
  request.log.error({ err: error, requestId }, 'Unexpected error')
  reply.status(500).send({
    code:      'INTERNAL_ERROR',
    message:   'Internal server error',
    requestId
  })
})
```

## Result 타입 패턴

```typescript
type Result<T, E = AppError> =
  | { ok: true;  value: T }
  | { ok: false; error: E }

function ok<T>(value: T): Result<T> {
  return { ok: true, value }
}

function err<E extends AppError>(error: E): Result<never, E> {
  return { ok: false, error }
}

// 사용 예
async function createUser(email: string): Promise<Result<User>> {
  const existing = await db.select().from(users).where(eq(users.email, email))
  if (existing.length > 0) return err(new ConflictError('Email already exists'))
  
  const [user] = await db.insert(users).values({ email }).returning()
  return ok(user)
}

// 컨트롤러에서 처리
const result = await createUser(body.email)
if (!result.ok) throw result.error  // Fastify가 에러 핸들러로 위임
return result.value
```

## 에러 재시도 패턴

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options = { maxAttempts: 3, delayMs: 1000, backoff: 2 }
): Promise<T> {
  let lastError: Error
  
  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err as Error
      
      if (attempt === options.maxAttempts) break
      
      const delay = options.delayMs * Math.pow(options.backoff, attempt - 1)
      await new Promise(resolve => setTimeout(resolve, delay + Math.random() * 100))
    }
  }
  
  throw lastError!
}
```

## 체크리스트
- [ ] 모든 에러에 고유 code 부여
- [ ] 5xx 에러는 requestId 포함하여 추적 가능하게
- [ ] 스택 트레이스는 프로덕션 응답에 노출 금지
- [ ] 에러 로그에 충분한 컨텍스트 포함 (userId, requestId)
- [ ] DB/외부 API 호출에 타임아웃 설정
