# Networking

## HTTP 클라이언트 (undici)

```typescript
import { fetch, Agent } from 'undici'

// 연결 풀 설정
const agent = new Agent({
  connections:            50,
  pipelining:             1,
  keepAliveTimeout:       30000,
  keepAliveMaxTimeout:    300000,
  connectTimeout:         5000,
  headersTimeout:         10000,
  bodyTimeout:            30000
})

// 타임아웃 포함 요청
async function httpGet<T>(url: string, options?: RequestInit): Promise<T> {
  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), 10000)

  try {
    const response = await fetch(url, {
      ...options,
      signal:    controller.signal,
      dispatcher: agent
    })

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${url}`)
    }

    return response.json() as Promise<T>
  } finally {
    clearTimeout(timeout)
  }
}
```

## 재시도 및 Circuit Breaker

```typescript
import retry from 'async-retry'

async function fetchWithRetry<T>(url: string): Promise<T> {
  return retry(
    async (bail, attempt) => {
      try {
        return await httpGet<T>(url)
      } catch (err) {
        // 4xx 에러는 재시도 불필요
        if (err instanceof Error && err.message.includes('HTTP 4')) {
          bail(err)
          return
        }
        throw err
      }
    },
    {
      retries:   3,
      factor:    2,
      minTimeout: 1000,
      maxTimeout: 10000,
      onRetry:   (err, attempt) => logger.warn({ err, attempt }, 'Retrying request')
    }
  )
}
```

## WebSocket (Fastify)

```typescript
import websocket from '@fastify/websocket'

await app.register(websocket)

app.get('/ws', { websocket: true }, (socket, req) => {
  socket.on('message', async (message) => {
    try {
      const data = JSON.parse(message.toString())
      const result = await processMessage(data)
      socket.send(JSON.stringify({ type: 'response', data: result }))
    } catch (err) {
      socket.send(JSON.stringify({ type: 'error', message: err.message }))
    }
  })

  socket.on('close', () => {
    logger.info({ userId: req.user?.id }, 'WebSocket closed')
  })

  // 핑/퐁으로 연결 유지
  const pingInterval = setInterval(() => {
    if (socket.readyState === socket.OPEN) socket.ping()
  }, 30000)

  socket.on('close', () => clearInterval(pingInterval))
})
```

## gRPC (서비스 간 통신)

```typescript
import * as grpc from '@grpc/grpc-js'
import * as protoLoader from '@grpc/proto-loader'

const packageDef = protoLoader.loadSync('protos/user.proto', {
  keepCase:  true,
  longs:     String,
  enums:     String,
  defaults:  true,
  oneofs:    true
})

const proto = grpc.loadPackageDefinition(packageDef) as any

const client = new proto.UserService(
  'user-service:50051',
  grpc.credentials.createInsecure(),
  { 'grpc.max_receive_message_length': 10 * 1024 * 1024 }
)

// Promise 래퍼
function getUser(id: string): Promise<UserResponse> {
  return new Promise((resolve, reject) => {
    client.GetUser({ id }, (err: any, response: any) => {
      if (err) reject(err)
      else resolve(response)
    })
  })
}
```

## 네트워크 보안

```typescript
// SSRF 방지 - 내부 IP 차단
import ipaddr from 'ipaddr.js'

function isPrivateIP(ip: string): boolean {
  try {
    const addr = ipaddr.parse(ip)
    return addr.range() !== 'unicast'
  } catch {
    return true  // 파싱 실패 = 차단
  }
}

async function safeHttpGet(url: string) {
  const { hostname } = new URL(url)
  const resolved = await dns.promises.lookup(hostname)
  if (isPrivateIP(resolved.address)) throw new ForbiddenError('Private IP not allowed')
  return httpGet(url)
}
```

## 체크리스트
- [ ] 모든 외부 HTTP 요청에 타임아웃 설정
- [ ] 연결 풀 사용 (연결 재사용)
- [ ] 재시도 로직에 지수 백오프 적용
- [ ] SSRF 방지 (내부 IP 차단)
- [ ] TLS 인증서 검증 비활성화 금지
