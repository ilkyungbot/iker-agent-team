# Build Optimization — 빌드 최적화

> 빌드 시간과 번들 크기는 개발자 경험과 사용자 경험 모두에 직결된다.

## 핵심 원칙
- **번들 분석 주기적 실행**: 크기가 갑자기 늘면 즉시 원인 파악
- **트리쉐이킹**: 사용하지 않는 코드는 번들에서 제거
- **코드 스플리팅**: 초기 로드에 필요한 코드만
- **캐싱 극대화**: 변경 없는 파일은 브라우저/CDN 캐시 활용

## 도구
| 목적 | 도구 |
|------|------|
| 번들 분석 | @next/bundle-analyzer |
| 크기 확인 | bundlephobia.com |
| 성능 측정 | Lighthouse, WebPageTest |
| 이미지 최적화 | next/image, sharp |

## 코드 예시

✅ Bundle Analyzer 설정
```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

module.exports = withBundleAnalyzer({
  // next config
})

// package.json
// "analyze": "ANALYZE=true next build"
```

✅ Dynamic Import로 코드 스플리팅
```typescript
import dynamic from 'next/dynamic'

// 무거운 에디터 컴포넌트
const RichTextEditor = dynamic(
  () => import('@/features/editor/RichTextEditor'),
  {
    loading: () => <EditorSkeleton />,
    ssr: false,
  }
)

// 조건부 기능
const DevTools = dynamic(
  () => import('@/shared/DevTools'),
  { ssr: false }
)

// 개발환경에서만
{process.env.NODE_ENV === 'development' && <DevTools />}
```

✅ 이미지 최적화 설정
```javascript
// next.config.js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.example.com' },
    ],
    deviceSizes: [640, 750, 828, 1080, 1200],
  },
}
```

✅ 폰트 최적화
```typescript
// app/layout.tsx
import { Inter, Noto_Sans_KR } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
})

const notoSansKR = Noto_Sans_KR({
  subsets: ['latin'],
  weight: ['400', '500', '700'],
  display: 'swap',
  variable: '--font-noto-sans-kr',
  preload: false, // 큰 폰트는 preload 주의
})
```

✅ 환경별 설정 최적화
```javascript
// next.config.js
module.exports = {
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
  experimental: {
    optimizeCss: true, // CSS 인라이닝 최적화
  },
}
```

## 실전 팁
- `sideEffects: false` in package.json으로 트리쉐이킹 힌트 제공
- `turbopack` (Next.js 15+) 개발 서버 속도 향상
- `.next/cache` CI 캐싱으로 빌드 시간 단축
- `next build && next start` 로컬에서 프로덕션 성능 검증

## 주의사항
- barrel 파일 (`index.ts`) 과도 사용 시 트리쉐이킹 방해
- moment.js 포함 시 번들 300KB+ 급증 → date-fns로 교체
- 폰트 preload 남용 → 불필요한 네트워크 요청
