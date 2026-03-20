# Analytics — 분석 및 트래킹

> 측정하지 않으면 개선할 수 없다. 하지만 모든 것을 측정하면 아무것도 개선되지 않는다.

## 핵심 원칙
- **의미 있는 이벤트만**: 비즈니스 의사결정에 필요한 이벤트만 수집
- **타입 안전 트래킹**: 이벤트 이름과 속성을 타입으로 정의
- **프라이버시 우선**: GDPR/개인정보보호법 준수, 동의 후 수집
- **성능 영향 최소화**: async 로딩, 트래킹 실패가 기능에 영향 없어야 함

## 코드 예시

✅ 타입 안전 이벤트 정의
```typescript
// shared/analytics/events.ts
interface AnalyticsEvents {
  // 버튼 클릭
  button_click: {
    button_name: string
    page: string
    section?: string
  }
  // 폼 제출
  form_submit: {
    form_name: string
    success: boolean
    error_code?: string
  }
  // 페이지 뷰 (자동 수집 제외 수동 트래킹)
  page_view: {
    page_path: string
    page_title: string
    referrer?: string
  }
  // 구매/전환
  purchase: {
    product_id: string
    amount: number
    currency: string
  }
}

type EventName = keyof AnalyticsEvents
```

✅ Analytics 클래스 구현
```typescript
class Analytics {
  private enabled = typeof window !== 'undefined' && !!window.gtag

  track<E extends EventName>(event: E, properties: AnalyticsEvents[E]) {
    if (!this.enabled) return

    try {
      window.gtag('event', event, properties)
      // Mixpanel, Amplitude 등 추가 가능
    } catch (error) {
      console.error('Analytics error:', error)
      // 트래킹 실패는 조용히 처리
    }
  }

  page(path: string, title: string) {
    this.track('page_view', { page_path: path, page_title: title })
  }
}

export const analytics = new Analytics()
```

✅ 커스텀 훅으로 추상화
```typescript
function useTrackEvent() {
  return useCallback(
    <E extends EventName>(event: E, properties: AnalyticsEvents[E]) => {
      analytics.track(event, properties)
    },
    []
  )
}

// 컴포넌트에서
function BuyButton({ productId, amount }: Props) {
  const track = useTrackEvent()

  return (
    <button
      onClick={() => {
        track('purchase', { product_id: productId, amount, currency: 'KRW' })
        handlePurchase()
      }}
    >
      구매하기
    </button>
  )
}
```

✅ Next.js Route Change 트래킹
```typescript
// app/layout.tsx or providers
'use client'
import { usePathname } from 'next/navigation'
import { useEffect } from 'react'

function AnalyticsProvider({ children }: { children: React.ReactNode }) {
  const pathname = usePathname()

  useEffect(() => {
    analytics.page(pathname, document.title)
  }, [pathname])

  return <>{children}</>
}
```

## 실전 팁
- `data-analytics` 속성으로 선언적 트래킹 구현 가능
- A/B 테스트 결과는 analytics 이벤트에 variant 속성 포함
- 개발 환경에서 analytics 이벤트 콘솔 출력으로 검증

## 주의사항
- PII(개인식별정보) 절대 이벤트 속성에 포함 금지
- 트래킹 코드가 핵심 기능 실행을 블로킹하면 안 됨
- 쿠키 동의 없이 트래킹 쿠키 설정 금지 (GDPR)
