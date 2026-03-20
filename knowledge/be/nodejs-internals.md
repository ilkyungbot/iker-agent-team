# Node.js Internals

## 이벤트 루프 이해

```
┌───────────────────────────┐
│           timers          │  setTimeout, setInterval
├───────────────────────────┤
│     pending callbacks     │  이전 루프 I/O 에러
├───────────────────────────┤
│       idle, prepare       │  내부 사용
├───────────────────────────┤
│           poll            │  I/O 이벤트 대기/실행 ← 주요 단계
├───────────────────────────┤
│           check           │  setImmediate
├───────────────────────────┤
│      close callbacks      │  socket.on('close')
└───────────────────────────┘
```

## Event Loop 블로킹 방지

```typescript
// 위험: CPU 집약적 작업이 루프를 블로킹
app.get('/compute', async () => {
  const result = heavyComputation()  // 동기 블로킹!
  return result
})

// 해결 1: Worker Threads (CPU 바운드)
import { Worker, workerData, parentPort } from 'worker_threads'

function runInWorker<T>(workerScript: string, data: unknown): Promise<T> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(workerScript, { workerData: data })
    worker.on('message', resolve)
    worker.on('error', reject)
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`))
    })
  })
}

// 해결 2: setImmediate로 청크 분할 (중간 크기 작업)
async function processLargeArray(items: unknown[]) {
  const CHUNK_SIZE = 1000
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE)
    await processChunk(chunk)
    await new Promise(resolve => setImmediate(resolve))  // 루프 양보
  }
}
```

## Worker Threads 풀

```typescript
import Piscina from 'piscina'

const pool = new Piscina({
  filename:    new URL('./worker.js', import.meta.url).href,
  maxThreads:  4,
  minThreads:  2,
  idleTimeout: 60000
})

// worker.js
module.exports = async ({ data }) => {
  return heavyComputation(data)
}

// 메인 스레드
const result = await pool.run({ data: inputData })
```

## 메모리 관리

```typescript
// Heap 사용량 모니터링
const used = process.memoryUsage()
console.log({
  heapUsed:  `${Math.round(used.heapUsed / 1024 / 1024)}MB`,
  heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)}MB`,
  external:  `${Math.round(used.external / 1024 / 1024)}MB`,
  rss:       `${Math.round(used.rss / 1024 / 1024)}MB`
})

// WeakRef / WeakMap으로 메모리 누수 방지
const cache = new WeakMap<object, ComputedResult>()

// 가비지 컬렉션 힌트 (Node.js 22+)
// global.gc?.()  // --expose-gc 플래그 필요
```

## 비동기 컨텍스트 추적 (AsyncLocalStorage)

```typescript
import { AsyncLocalStorage } from 'async_hooks'

interface RequestContext {
  requestId: string
  userId?:   string
  startTime: number
}

export const requestContext = new AsyncLocalStorage<RequestContext>()

// Fastify 훅에서 컨텍스트 설정
app.addHook('onRequest', async (req) => {
  requestContext.run(
    { requestId: req.id, startTime: Date.now() },
    () => {}
  )
})

// 서비스 레이어에서 접근
function getCurrentRequestId() {
  return requestContext.getStore()?.requestId ?? 'unknown'
}
```

## 프로세스 이벤트 처리

```typescript
// Graceful shutdown
const shutdown = async (signal: string) => {
  logger.info({ signal }, 'Received shutdown signal')
  
  await app.close()          // Fastify 서버 종료
  await pool.end()           // DB 연결 풀 종료
  await redisClient.quit()   // Redis 연결 종료
  
  process.exit(0)
}

process.on('SIGTERM', () => shutdown('SIGTERM'))
process.on('SIGINT',  () => shutdown('SIGINT'))

// 처리되지 않은 예외
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception')
  process.exit(1)
})

process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled promise rejection')
  process.exit(1)
})
```

## 체크리스트
- [ ] CPU 집약적 작업은 Worker Threads로 분리
- [ ] 이벤트 리스너 반드시 cleanup
- [ ] AsyncLocalStorage로 요청 컨텍스트 전파
- [ ] 프로세스 시그널 핸들러 등록 (SIGTERM/SIGINT)
- [ ] 메모리 사용량 주기적 모니터링
