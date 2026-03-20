# Performance — 프론트엔드 성능 최적화

> 측정 없이 최적화하지 마라. 추측이 아닌 데이터로 판단하라.

## 핵심 원칙
- **측정 먼저**: Chrome DevTools, Lighthouse, Web Vitals로 병목 확인 후 최적화
- **Core Web Vitals**: LCP < 2.5s, INP < 200ms, CLS < 0.1
- **번들 크기**: 초기 JS < 200KB (gzipped), 코드 스플리팅으로 레이지 로딩
- **렌더 최적화**: 불필요한 리렌더 방지가 메모이제이션보다 우선

## 최적화 체크리스트
| 영역 | 기법 |
|------|------|
| 번들 | Dynamic import, Tree shaking, 번들 분석 |
| 이미지 | Next.js Image, WebP, lazy loading, srcset |
| 렌더링 | React.memo, useMemo, useCallback (필요할 때만) |
| 네트워크 | 캐싱, 프리페칭, 스트리밍 |
| 폰트 | font-display: swap, 서브셋, self-hosting |

## 코드 예시

❌ 모든 것을 메모이제이션
```typescript
// 값이 단순하고 계산 비용이 낮으면 오히려 손해
const name = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName])
const handleClick = useCallback(() => setCount(c => c + 1), [])
```

✅ 실제로 비싼 연산에만 메모이제이션
```typescript
// 복잡한 필터링/정렬이 있을 때만
const sortedFilteredItems = useMemo(
  () => items
    .filter(item => item.category === selectedCategory)
    .sort((a, b) => b.createdAt - a.createdAt),
  [items, selectedCategory]
)
```

✅ 코드 스플리팅
```typescript
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // 클라이언트 전용 라이브러리
})
```

✅ 이미지 최적화
```tsx
import Image from 'next/image'

<Image
  src="/hero.jpg"
  alt="히어로 이미지"
  width={1200}
  height={630}
  priority // LCP 이미지에만
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
/>
```

✅ 가상화 (긴 목록)
```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null)
  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,
  })
  // ...
}
```

## 실전 팁
- `next/bundle-analyzer`로 번들 시각화 — 큰 덩어리 찾기
- React DevTools Profiler로 리렌더 원인 추적
- `requestIdleCallback`으로 비중요 작업 지연
- Intersection Observer로 뷰포트 밖 컴포넌트 지연 로딩

## 주의사항
- premature optimization: 실제 문제가 생기기 전에 최적화 X
- useCallback/useMemo 남용: 클로저 생성 비용 > 최적화 이득일 수 있음
- 서버 컴포넌트로 JS 번들 자체를 줄이는 게 클라이언트 최적화보다 효과적
