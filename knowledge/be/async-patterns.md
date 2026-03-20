# Async Patterns

## Promise 조합 패턴

```typescript
// 병렬 실행 (독립적인 작업)
const [user, orders, settings] = await Promise.all([
  userService.findById(userId),
  orderService.findByUser(userId),
  settingsService.findByUser(userId)
])

// 부분 실패 허용 (allSettled)
const results = await Promise.allSettled([
  notificationService.send(email1),
  notificationService.send(email2),
  notificationService.send(email3)
])

const failed = results
  .filter((r): r is PromiseRejectedResult => r.status === 'rejected')
  .map(r => r.reason)

if (failed.length > 0) logger.warn({ failed }, 'Some notifications failed')

// 첫 번째 성공 (경쟁)
const result = await Promise.any([
  fetchFromPrimaryDB(),
  fetchFromReplicaDB()
])

// 타임아웃 경쟁
const withTimeout = <T>(promise: Promise<T>, ms: number): Promise<T> => {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  )
  return Promise.race([promise, timeout])
}
```

## 비동기 큐 패턴 (동시성 제한)

```typescript
import PQueue from 'p-queue'

// 동시 실행 3개 제한
const queue = new PQueue({ concurrency: 3 })

async function processItems(items: Item[]) {
  const results = await Promise.all(
    items.map(item => queue.add(() => processItem(item)))
  )
  return results
}

// Rate limiting
const rateLimitedQueue = new PQueue({
  concurrency:          1,
  interval:             1000,
  intervalCap:          10  // 초당 10개
})
```

## 제너레이터 기반 스트리밍

```typescript
async function* streamDBRows(query: SelectQueryBuilder) {
  const BATCH_SIZE = 100
  let offset = 0

  while (true) {
    const rows = await query.limit(BATCH_SIZE).offset(offset)
    if (rows.length === 0) break

    yield* rows
    if (rows.length < BATCH_SIZE) break
    offset += BATCH_SIZE
  }
}

// 사용
for await (const row of streamDBRows(db.select().from(orders))) {
  await processRow(row)
}
```

## Circuit Breaker 패턴

```typescript
enum CircuitState { CLOSED, OPEN, HALF_OPEN }

class CircuitBreaker {
  private state       = CircuitState.CLOSED
  private failures    = 0
  private lastFailure = 0

  constructor(
    private threshold:  number = 5,
    private resetMs:    number = 60000
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailure > this.resetMs) {
        this.state = CircuitState.HALF_OPEN
      } else {
        throw new Error('Circuit breaker is OPEN')
      }
    }

    try {
      const result = await fn()
      this.onSuccess()
      return result
    } catch (err) {
      this.onFailure()
      throw err
    }
  }

  private onSuccess() {
    this.failures = 0
    this.state    = CircuitState.CLOSED
  }

  private onFailure() {
    this.failures++
    this.lastFailure = Date.now()
    if (this.failures >= this.threshold) {
      this.state = CircuitState.OPEN
    }
  }
}
```

## 비동기 이벤트 처리 (EventEmitter + AsyncIterator)

```typescript
import { on } from 'events'
import { EventEmitter } from 'events'

const emitter = new EventEmitter()

// AsyncIterator로 이벤트 스트림 소비
async function consumeEvents() {
  for await (const [event] of on(emitter, 'data')) {
    await processEvent(event)
  }
}

// AbortSignal로 정리
async function consumeWithCleanup(signal: AbortSignal) {
  for await (const [event] of on(emitter, 'data', { signal })) {
    await processEvent(event)
  }
}
```

## 체크리스트
- [ ] 독립적 작업은 Promise.all로 병렬화
- [ ] 외부 API 호출에 타임아웃 설정
- [ ] 무제한 동시성 방지 (p-queue 또는 semaphore)
- [ ] 불안정한 외부 서비스에 Circuit Breaker 적용
- [ ] 제너레이터로 대용량 데이터 스트리밍 처리
