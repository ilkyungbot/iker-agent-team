# Styling — 스타일링 전략

> 스타일은 코드다. 일관성, 유지보수성, 성능 모두 고려해야 한다.

## 핵심 원칙
- **Tailwind 유틸리티 우선**: 컴포넌트 단위 인라인 스타일
- **토큰 기반**: 색상, 간격, 타이포그래피는 CSS 변수
- **반응형 모바일 우선**: `sm:`, `md:`, `lg:` 순서로
- **다크 모드**: `dark:` 변형으로 시스템 테마 지원

## 기술 스택
- **Tailwind CSS v3**: 유틸리티 퍼스트
- **CSS Variables**: 테마 토큰
- **CVA**: 컴포넌트 변형 관리
- **tailwind-merge**: 클래스 충돌 해결

## 코드 예시

✅ cn() 유틸 함수
```typescript
// shared/lib/utils.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

// 사용
<div className={cn(
  'base-styles',
  isActive && 'active-styles',
  className // 외부에서 전달된 클래스
)} />
```

✅ CSS Variables 테마 설정
```css
/* globals.css */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --primary: 222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;
  --border: 214.3 31.8% 91.4%;
  --radius: 0.5rem;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  --primary: 210 40% 98%;
  --primary-foreground: 222.2 47.4% 11.2%;
}
```

✅ 반응형 레이아웃
```tsx
<div className="
  grid
  grid-cols-1
  sm:grid-cols-2
  lg:grid-cols-3
  xl:grid-cols-4
  gap-4
  p-4
  md:p-6
  lg:p-8
">
```

✅ 복잡한 애니메이션 (Framer Motion)
```typescript
import { motion, AnimatePresence } from 'framer-motion'

function Notification({ message, isVisible }: Props) {
  return (
    <AnimatePresence>
      {isVisible && (
        <motion.div
          initial={{ opacity: 0, y: -20 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -20 }}
          transition={{ duration: 0.2 }}
        >
          {message}
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

❌ 인라인 스타일로 토큰 우회
```tsx
<div style={{ color: '#6366f1', marginTop: '16px' }}>
```

✅ Tailwind 토큰 활용
```tsx
<div className="text-primary mt-4">
```

## 실전 팁
- `prettier-plugin-tailwindcss`로 클래스 자동 정렬
- `tailwind-merge`로 동적 클래스 충돌 방지
- `@layer components`로 자주 쓰는 패턴 추출

## 주의사항
- Tailwind arbitrary value 남용 → 커스텀 토큰 추가로 해결
- CSS-in-JS (styled-components) : SSR 성능 문제, Tailwind와 혼용 지양
- `!important` 금지 — 특이도 문제는 구조 개선으로 해결
