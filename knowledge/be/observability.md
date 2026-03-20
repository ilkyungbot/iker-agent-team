# Observability

## 3기둥: Logs + Metrics + Traces

### 구조화 로깅 (Pino)

```typescript
import pino from 'pino'

export const logger = pino({
  level:     process.env.LOG_LEVEL ?? 'info',
  redact:    ['req.headers.authorization', 'body.password', 'body.token'],
  serializers: {
    req:  pino.stdSerializers.req,
    res:  pino.stdSerializers.res,
    err:  pino.stdSerializers.err
  },
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined
})
```

### 메트릭 (Prometheus)

```typescript
import { register, Counter, Histogram, Gauge } from 'prom-client'
import collectDefaultMetrics from 'prom-client'

collectDefaultMetrics()  // Node.js 기본 메트릭

export const httpRequestDuration = new Histogram({
  name:        'http_request_duration_seconds',
  help:        'HTTP request duration in seconds',
  labelNames:  ['method', 'route', 'status_code'],
  buckets:     [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
})

export const httpRequestTotal = new Counter({
  name:       'http_requests_total',
  help:       'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code']
})

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
})

// Fastify 훅으로 자동 수집
app.addHook('onResponse', async (req, reply) => {
  const duration = reply.elapsedTime / 1000
  const labels = { method: req.method, route: req.routerPath, status_code: reply.statusCode }
  httpRequestDuration.observe(labels, duration)
  httpRequestTotal.inc(labels)
})

// /metrics 엔드포인트
app.get('/metrics', async (_, reply) => {
  reply.type(register.contentType)
  return register.metrics()
})
```

### 분산 추적 (OpenTelemetry)

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { Resource } from '@opentelemetry/resources'
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions'

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]:    'user-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION ?? '0.0.0',
    environment: process.env.NODE_ENV
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT
  })
})

sdk.start()
process.on('SIGTERM', () => sdk.shutdown())
```

## 헬스체크 엔드포인트

```typescript
app.get('/health', async (req, reply) => {
  const checks = await Promise.allSettled([
    db.execute(sql`SELECT 1`),
    redis.ping()
  ])

  const [dbCheck, redisCheck] = checks
  const healthy = checks.every(c => c.status === 'fulfilled')

  reply.status(healthy ? 200 : 503).send({
    status:    healthy ? 'healthy' : 'degraded',
    timestamp: new Date().toISOString(),
    checks: {
      database: dbCheck.status === 'fulfilled' ? 'ok' : 'error',
      redis:    redisCheck.status === 'fulfilled' ? 'ok' : 'error'
    }
  })
})

// Kubernetes readiness/liveness
app.get('/ready', async (_, reply) => reply.send({ status: 'ready' }))
app.get('/live',  async (_, reply) => reply.send({ status: 'alive' }))
```

## Alert 기준 (SLO 기반)

| 메트릭 | 경고 | 심각 |
|--------|------|------|
| 에러율 | > 1% | > 5% |
| P99 지연 | > 1s | > 3s |
| 가용성 | < 99.9% | < 99% |

## 체크리스트
- [ ] 모든 로그에 requestId 포함
- [ ] /metrics 엔드포인트 Prometheus 형식 제공
- [ ] /health 엔드포인트 DB/Redis 포함 체크
- [ ] 분산 추적 TraceId 로그에 포함
- [ ] 에러율 알림 설정
