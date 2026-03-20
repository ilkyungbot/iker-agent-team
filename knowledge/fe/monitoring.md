# Monitoring — 프론트엔드 모니터링

> 프로덕션은 스테이징이 아니다. 실제 사용자가 겪는 문제만 모니터링으로 알 수 있다.

## 핵심 원칙
- **에러 모니터링**: Sentry로 JS 에러 수집 및 알림
- **성능 모니터링**: Core Web Vitals 실측 데이터(RUM) 수집
- **사용자 세션**: 재현 가능한 버그 추적
- **알림 피로 방지**: 중요한 에러만 즉시 알림

## 도구 스택
| 목적 | 도구 |
|------|------|
| 에러 트래킹 | Sentry |
| 성능 RUM | Vercel Analytics, Sentry Performance |
| 로그 | Datadog, LogRocket |
| 알림 | Slack 연동 |

## 코드 예시

✅ Sentry 초기화 (Next.js)
```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true, // PII 보호
      blockAllMedia: false,
    }),
  ],
  beforeSend(event) {
    // 특정 에러 필터링
    if (event.exception?.values?.[0]?.type === 'ChunkLoadError') {
      return null // 청크 로드 에러는 무시 (배포 중 발생)
    }
    return event
  },
})
```

✅ 컨텍스트 추가
```typescript
// 사용자 식별 (로그인 후)
Sentry.setUser({
  id: user.id,
  email: user.email, // GDPR 주의
})

// 에러 캡처 with 컨텍스트
try {
  await processPayment(order)
} catch (error) {
  Sentry.captureException(error, {
    tags: { feature: 'payment', orderId: order.id },
    extra: { orderAmount: order.amount },
  })
  throw error
}
```

✅ Core Web Vitals 수집
```typescript
// app/layout.tsx
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals'

function reportWebVitals(metric: Metric) {
  // Vercel Analytics
  if (window.va) {
    window.va('event', metric.name, { value: metric.value })
  }

  // 직접 수집
  analytics.track('web_vital', {
    name: metric.name,
    value: Math.round(metric.value),
    rating: metric.rating, // 'good' | 'needs-improvement' | 'poor'
  })
}

onCLS(reportWebVitals)
onINP(reportWebVitals)
onLCP(reportWebVitals)
onFCP(reportWebVitals)
onTTFB(reportWebVitals)
```

✅ 커스텀 성능 마킹
```typescript
function measureApiCall(name: string, fn: () => Promise<unknown>) {
  return async () => {
    const start = performance.now()
    try {
      return await fn()
    } finally {
      const duration = performance.now() - start
      performance.measure(name, { duration })
      if (duration > 3000) {
        Sentry.captureMessage(`Slow API: ${name} took ${duration}ms`, 'warning')
      }
    }
  }
}
```

## 실전 팁
- Sentry release tracking으로 에러가 어떤 배포에서 발생했는지 추적
- Source maps 업로드로 minified 코드도 디버깅 가능
- 알림 임계값 설정 — 1시간에 10회 이상 에러만 Slack 알림

## 주의사항
- `console.error` != Sentry 수집 — 명시적 `captureException` 필요
- 샘플링 너무 높으면 비용 폭증 — 프로덕션 tracesSampleRate 0.1 이하
- 에러 무한 반복 알림 방지 — Sentry issue grouping 설정
