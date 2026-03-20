# Rate Limiting

## 알고리즘 비교

| 알고리즘 | 특징 | 사용 사례 |
|---------|------|-----------|
| Fixed Window | 단순, 경계 버스트 발생 | 단순 API 보호 |
| Sliding Window | 정확, 메모리 더 사용 | 일반 API |
| Token Bucket | 버스트 허용, 유연 | 유연한 제한 |
| Leaky Bucket | 균일한 처리율 | 큐잉 시스템 |

## @fastify/rate-limit 설정

```typescript
import rateLimit from '@fastify/rate-limit'

// 전역 설정
await app.register(rateLimit, {
  global:     true,
  max:        100,
  timeWindow: '1 minute',
  redis:      redis,          // 분산 환경
  keyGenerator: (req) => req.user?.sub ?? req.ip,  // 사용자 기반
  errorResponseBuilder: (req, context) => ({
    code:       'RATE_LIMIT_EXCEEDED',
    message:    `Too many requests. Try again in ${context.after}`,
    retryAfter: context.after
  })
})
```

## 엔드포인트별 세분화

```typescript
// 로그인: 매우 엄격
app.post('/api/v1/auth/login', {
  config: {
    rateLimit: {
      max:        5,
      timeWindow: '15 minutes',
      keyGenerator: (req) => req.ip  // IP 기반 (인증 전)
    }
  },
  handler: loginHandler
})

// 일반 API: 관대
app.get('/api/v1/products', {
  config: {
    rateLimit: {
      max:        1000,
      timeWindow: '1 minute'
    }
  },
  handler: listProductsHandler
})

// 파일 업로드: 제한적
app.post('/api/v1/files', {
  config: {
    rateLimit: {
      max:        10,
      timeWindow: '1 hour',
      keyGenerator: (req) => req.user?.sub ?? req.ip
    }
  }
})
```

## Redis 기반 슬라이딩 윈도우

```typescript
const RATE_LIMIT_SCRIPT = `
  local key        = KEYS[1]
  local now        = tonumber(ARGV[1])
  local window     = tonumber(ARGV[2])
  local max        = tonumber(ARGV[3])
  local windowStart = now - window

  redis.call('ZREMRANGEBYSCORE', key, 0, windowStart)
  local count = redis.call('ZCARD', key)

  if count < max then
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('EXPIRE', key, window / 1000)
    return {1, max - count - 1}
  end

  return {0, 0}
`

async function checkRateLimit(
  key:    string,
  max:    number,
  windowMs: number
): Promise<{ allowed: boolean; remaining: number }> {
  const result = await redis.eval(
    RATE_LIMIT_SCRIPT,
    1, key,
    Date.now().toString(),
    windowMs.toString(),
    max.toString()
  ) as [number, number]

  return { allowed: result[0] === 1, remaining: result[1] }
}
```

## 응답 헤더

```typescript
// 표준 Rate Limit 헤더
reply.header('X-RateLimit-Limit',     maxRequests.toString())
reply.header('X-RateLimit-Remaining', remaining.toString())
reply.header('X-RateLimit-Reset',     resetTimestamp.toString())
reply.header('Retry-After',           retryAfterSeconds.toString())
```

## IP 화이트리스트

```typescript
const WHITELIST = new Set([
  '127.0.0.1',
  '10.0.0.0/8',        // 내부 네트워크
  process.env.ADMIN_IP
].filter(Boolean))

function isWhitelisted(ip: string): boolean {
  return WHITELIST.has(ip)
}

await app.register(rateLimit, {
  skip: (req) => isWhitelisted(req.ip)
})
```

## 체크리스트
- [ ] 인증 엔드포인트에 엄격한 제한 (로그인/회원가입)
- [ ] Redis 기반 분산 카운터 (다중 서버 환경)
- [ ] 응답 헤더로 제한 정보 노출
- [ ] IP/사용자 기반 키 전략 선택
- [ ] 화이트리스트로 내부 서비스 제외
