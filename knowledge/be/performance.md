# Performance Optimization

## 프로파일링 먼저

```bash
# Node.js 내장 프로파일러
node --prof app.js
node --prof-process isolate-*.log > profile.txt

# clinic.js (시각적 분석)
npx clinic doctor -- node app.js
npx clinic flame  -- node app.js
```

## 쿼리 최적화

```typescript
// N+1 문제 해결 - DataLoader 패턴
import DataLoader from 'dataloader'

const userLoader = new DataLoader<string, User>(async (ids) => {
  const rows = await db.select().from(users).where(inArray(users.id, ids as string[]))
  const map = new Map(rows.map(u => [u.id, u]))
  return ids.map(id => map.get(id) ?? null)
})

// 사용
const user = await userLoader.load(userId)  // 자동 배칭

// Select 컬럼 명시 (SELECT * 금지)
const minimalUsers = await db.select({
  id:   users.id,
  name: users.name
}).from(users)
```

## 응답 압축

```typescript
import compress from '@fastify/compress'

await app.register(compress, {
  global:      true,
  encodings:   ['br', 'gzip'],
  threshold:   1024,           // 1KB 이상만 압축
  brotliOptions: { params: { [zlib.constants.BROTLI_PARAM_QUALITY]: 4 } }
})
```

## 연결 재사용

```typescript
// HTTP Keep-Alive (기본값이지만 명시)
const app = Fastify({ keepAliveTimeout: 72000 })

// undici (내장 HTTP 클라이언트) - 연결 풀
import { Pool } from 'undici'

const pool = new Pool('https://api.external.com', {
  connections:    10,
  pipelining:     1,
  keepAliveTimeout: 30000
})

const { body } = await pool.request({
  path:   '/endpoint',
  method: 'GET'
})
```

## 캐싱 전략 (→ caching.md 참조)

```typescript
// ETag 기반 조건부 요청
app.get('/users/:id', async (req, reply) => {
  const user = await getUser(req.params.id)
  const etag = `"${user.updatedAt.getTime()}"`
  
  if (req.headers['if-none-match'] === etag) {
    return reply.status(304).send()
  }
  
  reply.header('ETag', etag)
  reply.header('Cache-Control', 'private, max-age=60')
  return user
})
```

## 스트리밍 응답 (대용량 데이터)

```typescript
import { pipeline } from 'stream/promises'
import { stringify } from 'JSONStream'

app.get('/export/orders', async (req, reply) => {
  reply.type('application/json')
  
  const cursor = db.select().from(orders).cursor()
  const jsonStream = stringify('{"data": [', ',', ']}')
  
  reply.send(jsonStream)
  
  for await (const row of cursor) {
    jsonStream.write(row)
  }
  jsonStream.end()
})
```

## 메모리 관리

```typescript
// 메모리 누수 방지 패턴
class EventEmitterService {
  private emitter = new EventEmitter()
  
  subscribe(event: string, handler: Function) {
    this.emitter.on(event, handler)
    // cleanup 반환
    return () => this.emitter.off(event, handler)
  }
}

// Map/Set 크기 제한 (캐시 목적)
const cache = new Map<string, unknown>()
const MAX_SIZE = 1000

function setCache(key: string, value: unknown) {
  if (cache.size >= MAX_SIZE) {
    // LRU: 첫 번째 항목 제거
    cache.delete(cache.keys().next().value)
  }
  cache.set(key, value)
}
```

## 성능 지표 목표값

| 지표 | 목표 |
|------|------|
| API P95 응답시간 | < 200ms |
| API P99 응답시간 | < 500ms |
| DB 쿼리 평균 | < 50ms |
| 메모리 사용 | < 512MB |
| CPU 사용률 (idle) | < 20% |

## 체크리스트
- [ ] EXPLAIN ANALYZE로 슬로우 쿼리 확인
- [ ] N+1 쿼리 제거 (DataLoader or JOIN)
- [ ] 응답 압축 활성화
- [ ] 불필요한 SELECT * 제거
- [ ] 연결 풀 크기 적절히 설정
