# Caching

## 캐싱 전략

| 전략 | 설명 | 사용 사례 |
|------|------|-----------|
| Cache-Aside | 앱이 직접 캐시 관리 | 사용자 프로필, 상품 정보 |
| Write-Through | 쓰기 시 캐시+DB 동시 업데이트 | 실시간 카운터 |
| Write-Behind | 쓰기 시 캐시만, 비동기로 DB | 조회수, 좋아요 |
| Read-Through | 캐시 미스 시 자동으로 DB 조회 | ORM 레벨 캐싱 |

## Redis 클라이언트 설정

```typescript
import { Redis } from 'ioredis'

export const redis = new Redis({
  host:             process.env.REDIS_HOST,
  port:             Number(process.env.REDIS_PORT ?? 6379),
  password:         process.env.REDIS_PASSWORD,
  db:               0,
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect:      true,
  retryStrategy:    (times) => Math.min(times * 100, 3000)
})

redis.on('error', (err) => logger.error({ err }, 'Redis error'))
redis.on('connect', ()  => logger.info('Redis connected'))
```

## Cache-Aside 패턴

```typescript
const CACHE_TTL = 300  // 5분

async function getCachedUser(userId: string): Promise<User | null> {
  const key = `user:${userId}`
  
  // 1. 캐시 조회
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)
  
  // 2. DB 조회
  const user = await db.select().from(users).where(eq(users.id, userId)).limit(1)
  if (!user[0]) return null
  
  // 3. 캐시 저장
  await redis.setex(key, CACHE_TTL, JSON.stringify(user[0]))
  return user[0]
}

async function invalidateUserCache(userId: string) {
  await redis.del(`user:${userId}`)
}
```

## 캐시 유틸리티 클래스

```typescript
export class CacheService {
  constructor(private redis: Redis) {}

  async wrap<T>(
    key: string,
    ttlSeconds: number,
    fn: () => Promise<T>
  ): Promise<T> {
    const cached = await this.redis.get(key)
    if (cached) return JSON.parse(cached) as T

    const value = await fn()
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value))
    return value
  }

  async invalidate(pattern: string) {
    const keys = await this.redis.keys(pattern)
    if (keys.length > 0) {
      await this.redis.del(...keys)
    }
  }

  // 분산 캐시 워밍
  async warmup(keys: Array<{ key: string; ttl: number; fn: () => Promise<unknown> }>) {
    await Promise.allSettled(
      keys.map(({ key, ttl, fn }) => this.wrap(key, ttl, fn))
    )
  }
}
```

## HTTP 캐싱 (ETag / Cache-Control)

```typescript
app.get('/products/:id', async (req, reply) => {
  const product = await productService.findById(req.params.id)
  if (!product) throw new NotFoundError('Product', req.params.id)

  const etag = `"${product.updatedAt.getTime().toString(36)}"`
  
  // 304 Not Modified
  if (req.headers['if-none-match'] === etag) {
    return reply.status(304).send()
  }

  return reply
    .header('ETag', etag)
    .header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300')
    .send(product)
})
```

## 캐시 키 네이밍

```typescript
// 계층적 키 구조 (삭제 시 패턴 매칭 용이)
const keys = {
  user:        (id: string)          => `user:${id}`,
  userOrders:  (userId: string)      => `user:${userId}:orders`,
  product:     (id: string)          => `product:${id}`,
  productList: (page: number)        => `product:list:${page}`,
  session:     (token: string)       => `session:${token}`
}
```

## 체크리스트
- [ ] 캐시 TTL은 데이터 갱신 주기에 맞게 설정
- [ ] 쓰기 후 반드시 캐시 무효화
- [ ] Redis 다운 시 DB 폴백 처리
- [ ] 핫스팟 키 주의 (샤딩 또는 로컬 캐시 병행)
- [ ] 캐시 미스율 모니터링
