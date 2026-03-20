# Logging

## Pino 설정 및 사용

```typescript
import pino, { Logger } from 'pino'

const logger = pino({
  level:   process.env.LOG_LEVEL ?? 'info',
  base:    { service: 'api', version: process.env.APP_VERSION },
  redact: {
    paths:  ['req.headers.authorization', '*.password', '*.token', '*.secret'],
    censor: '[REDACTED]'
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level: (label) => ({ level: label })
  }
})

export { logger }
```

## 로그 레벨 가이드

| 레벨 | 사용 상황 | 예시 |
|------|-----------|------|
| `fatal` | 서비스 불능 상태 | DB 연결 실패, 시작 오류 |
| `error` | 예상치 못한 오류 | 처리 불가 예외, 5xx |
| `warn`  | 주의 필요, 정상 동작 | 재시도 발생, 느린 쿼리 |
| `info`  | 중요 비즈니스 이벤트 | 주문 생성, 결제 완료 |
| `debug` | 개발/디버깅용 | 함수 진입/종료, 변수 값 |
| `trace` | 최상세 추적 | 루프 반복, 저수준 I/O |

## 구조화 로깅 패턴

```typescript
// 항상 객체를 첫 번째 인자로
logger.info({ userId, orderId, amount }, 'Payment completed')

// 에러 로깅
logger.error({ err: error, userId, requestId }, 'Failed to process payment')

// 성능 측정
const start = Date.now()
try {
  const result = await doSomething()
  logger.info({ durationMs: Date.now() - start }, 'Operation completed')
  return result
} catch (err) {
  logger.error({ err, durationMs: Date.now() - start }, 'Operation failed')
  throw err
}
```

## Fastify 요청 로깅

```typescript
const app = Fastify({
  logger: {
    level:      process.env.LOG_LEVEL ?? 'info',
    redact:     ['req.headers.authorization'],
    serializers: {
      req(request) {
        return {
          method:    request.method,
          url:       request.url,
          requestId: request.id,
          userId:    request.user?.id  // 인증 후 가능
        }
      },
      res(reply) {
        return {
          statusCode: reply.statusCode,
          duration:   reply.elapsedTime
        }
      }
    }
  }
})

// 커스텀 요청 ID
app.addHook('onRequest', async (req) => {
  req.log = req.log.child({ requestId: req.id, traceId: getTraceId() })
})
```

## 자식 로거 (컨텍스트 전파)

```typescript
class UserService {
  private log: Logger

  constructor(logger: Logger) {
    this.log = logger.child({ service: 'UserService' })
  }

  async createUser(data: CreateUserDTO) {
    const log = this.log.child({ email: data.email })
    log.info('Creating user')
    
    try {
      const user = await this.repo.create(data)
      log.info({ userId: user.id }, 'User created successfully')
      return user
    } catch (err) {
      log.error({ err }, 'Failed to create user')
      throw err
    }
  }
}
```

## 로그 집계 설정

```yaml
# docker-compose.yml - Loki + Grafana
services:
  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
```

## 체크리스트
- [ ] 프로덕션: JSON 로그 (파싱 가능)
- [ ] 개발: pino-pretty (가독성)
- [ ] 모든 에러에 `err` 키로 Error 객체 포함
- [ ] 민감 정보 redact 설정
- [ ] 로그 레벨 환경변수로 동적 제어
- [ ] 느린 쿼리 (>100ms) warn 로깅
