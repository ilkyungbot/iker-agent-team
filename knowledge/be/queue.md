# Queue & Background Jobs

## BullMQ 설정

```typescript
import { Queue, Worker, QueueEvents } from 'bullmq'
import { Redis } from 'ioredis'

const connection = new Redis({
  host:                process.env.REDIS_HOST,
  port:                Number(process.env.REDIS_PORT ?? 6379),
  maxRetriesPerRequest: null  // BullMQ 필수 설정
})

// 큐 정의
export const emailQueue  = new Queue('email',  { connection })
export const reportQueue = new Queue('report', { connection })
```

## Job 타입 정의

```typescript
interface EmailJobData {
  to:      string
  subject: string
  template: 'welcome' | 'password-reset' | 'invoice'
  variables: Record<string, unknown>
}

interface ReportJobData {
  userId:   string
  reportId: string
  format:   'pdf' | 'xlsx'
}
```

## 워커 정의

```typescript
const emailWorker = new Worker<EmailJobData>(
  'email',
  async (job) => {
    const { to, subject, template, variables } = job.data
    
    await job.updateProgress(10)
    const html = await renderTemplate(template, variables)
    
    await job.updateProgress(50)
    await mailer.send({ to, subject, html })
    
    await job.updateProgress(100)
    return { sentAt: new Date().toISOString() }
  },
  {
    connection,
    concurrency: 5,
    limiter: {
      max:      100,
      duration: 60000  // 분당 100개
    }
  }
)

emailWorker.on('completed', (job, result) => {
  logger.info({ jobId: job.id, result }, 'Email sent')
})

emailWorker.on('failed', (job, err) => {
  logger.error({ jobId: job?.id, err }, 'Email job failed')
})
```

## Job 추가 패턴

```typescript
// 즉시 실행
await emailQueue.add('send-welcome', {
  to: user.email,
  subject: '가입을 환영합니다',
  template: 'welcome',
  variables: { name: user.name }
})

// 지연 실행 (5분 후)
await emailQueue.add('send-reminder', jobData, {
  delay: 5 * 60 * 1000
})

// 재시도 설정
await reportQueue.add('generate-report', jobData, {
  attempts:    3,
  backoff: {
    type:  'exponential',
    delay: 5000
  },
  removeOnComplete: { count: 100 },
  removeOnFail:     { count: 50 }
})

// 반복 작업 (Cron)
await reportQueue.add('daily-summary', {}, {
  repeat: { pattern: '0 9 * * *', tz: 'Asia/Seoul' }
})
```

## 우선순위 큐

```typescript
// priority: 높을수록 먼저 처리 (1~100)
await emailQueue.add('critical-alert', alertData, { priority: 100 })
await emailQueue.add('marketing',      promoData, { priority: 1   })
```

## 큐 모니터링

```typescript
const queueEvents = new QueueEvents('email', { connection })

queueEvents.on('waiting',   ({ jobId }) => metrics.increment('job.waiting'))
queueEvents.on('active',    ({ jobId }) => metrics.increment('job.active'))
queueEvents.on('completed', ({ jobId }) => metrics.increment('job.completed'))
queueEvents.on('failed',    ({ jobId, failedReason }) => {
  metrics.increment('job.failed')
  logger.warn({ jobId, failedReason }, 'Job failed')
})

// 큐 상태 확인
async function getQueueStats(queue: Queue) {
  const [waiting, active, completed, failed] = await Promise.all([
    queue.getWaitingCount(),
    queue.getActiveCount(),
    queue.getCompletedCount(),
    queue.getFailedCount()
  ])
  return { waiting, active, completed, failed }
}
```

## 체크리스트
- [ ] Redis `maxRetriesPerRequest: null` 설정 (BullMQ 요구사항)
- [ ] Worker concurrency는 CPU 코어 수 고려
- [ ] 실패 잡 재시도 횟수 및 backoff 설정
- [ ] 완료/실패 잡 자동 정리 설정 (removeOnComplete/Fail)
- [ ] Dead Letter Queue 패턴으로 영구 실패 잡 처리
