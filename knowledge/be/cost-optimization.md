# Cost Optimization

## 주요 비용 드라이버

| 항목 | 최적화 방향 |
|------|------------|
| DB 연산 | 쿼리 최적화, 읽기 복제본, 캐싱 |
| 네트워크 | 압축, CDN, 리전 최적화 |
| 컨테이너 | 적절한 리소스 설정, 오토스케일링 |
| 스토리지 | 계층화, 압축, 만료 정책 |
| 외부 API | 캐싱, 배칭, 구독 플랜 최적화 |

## DB 비용 최적화

```typescript
// 읽기/쓰기 분리
import { drizzle as drizzleWrite } from 'drizzle-orm/node-postgres'
import { drizzle as drizzleRead }  from 'drizzle-orm/node-postgres'

const writePool = new Pool({ connectionString: process.env.DB_WRITE_URL, max: 10 })
const readPool  = new Pool({ connectionString: process.env.DB_READ_URL,  max: 20 })

export const writeDb = drizzleWrite(writePool)
export const readDb  = drizzleRead(readPool)

// 목록 조회 → 읽기 복제본 사용
async function listUsers(query: ListUsersQuery) {
  return readDb.select().from(users).limit(query.limit)
}

// 쓰기 → 마스터 사용
async function createUser(data: CreateUserData) {
  return writeDb.insert(users).values(data).returning()
}
```

## 외부 API 배칭

```typescript
// Stripe 같은 API 비용 절감 - 배치 처리
class BatchProcessor<T, R> {
  private queue: Array<{ item: T; resolve: (r: R) => void; reject: (e: Error) => void }> = []
  private timer: NodeJS.Timeout | null = null

  constructor(
    private batchFn:  (items: T[]) => Promise<R[]>,
    private maxSize:  number = 100,
    private delayMs:  number = 50
  ) {}

  add(item: T): Promise<R> {
    return new Promise((resolve, reject) => {
      this.queue.push({ item, resolve, reject })

      if (this.queue.length >= this.maxSize) {
        this.flush()
      } else if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.delayMs)
      }
    })
  }

  private async flush() {
    if (this.timer) { clearTimeout(this.timer); this.timer = null }
    if (!this.queue.length) return

    const batch = this.queue.splice(0, this.maxSize)
    const results = await this.batchFn(batch.map(b => b.item))
    batch.forEach((b, i) => b.resolve(results[i]))
  }
}
```

## 클라우드 비용 모니터링

```typescript
// Cloud Cost 태그 전략
// 모든 리소스에 일관된 태그 적용
const resourceTags = {
  Environment: process.env.NODE_ENV,
  Service:     'api-server',
  Team:        'backend',
  CostCenter:  'engineering'
}

// 불필요한 로그 필터링 (CloudWatch/GCP 비용)
const logger = pino({
  level: process.env.NODE_ENV === 'production' ? 'warn' : 'debug',
  // 헬스체크 로그 제외 (노이즈 + 비용)
  hooks: {
    logMethod(args, method) {
      if (args[0]?.url === '/health') return
      method.apply(this, args)
    }
  }
})
```

## 오토스케일링 설정

```yaml
# Kubernetes HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type:               AverageUtilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type:               AverageUtilization
          averageUtilization: 80
```

## 체크리스트
- [ ] 읽기/쓰기 DB 분리 (읽기 복제본 활용)
- [ ] 외부 API 응답 캐싱 (비용 + 성능)
- [ ] 불필요한 로그 레벨 최적화
- [ ] 리소스 requests/limits 정기 검토
- [ ] 비용 태그 모든 리소스에 적용
- [ ] 사용하지 않는 인덱스 제거
