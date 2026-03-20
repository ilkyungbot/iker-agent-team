# Design System — 디자인 시스템 활용

> 컴포넌트는 디자인 시스템의 어휘다. 일관된 어휘가 일관된 제품을 만든다.

## 핵심 원칙
- **토큰 우선**: 하드코딩된 색상/간격 금지. 디자인 토큰 사용
- **컴포지션**: 작은 컴포넌트를 조합해 복잡한 UI 구성
- **변형(Variant) 명시**: `variant`, `size`, `intent` props로 외양 제어
- **shadcn/ui 기반**: 복사-붙여넣기 방식으로 커스터마이징 용이

## 기술 스택
- **shadcn/ui**: Radix UI + Tailwind CSS 기반 컴포넌트
- **Tailwind CSS**: 유틸리티 퍼스트 스타일링
- **CSS Variables**: 테마 토큰 관리
- **CVA (class-variance-authority)**: 컴포넌트 변형 관리

## 코드 예시

✅ CVA로 버튼 변형 관리
```typescript
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/shared/lib/utils'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'md',
    },
  }
)

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  isLoading?: boolean
}

export function Button({ variant, size, isLoading, children, className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      disabled={isLoading || props.disabled}
      {...props}
    >
      {isLoading ? <Spinner className="mr-2 h-4 w-4" /> : null}
      {children}
    </button>
  )
}
```

✅ 토큰 활용 (Tailwind 커스텀 설정)
```typescript
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        primary: 'hsl(var(--primary))',       // CSS 변수 참조
        'primary-foreground': 'hsl(var(--primary-foreground))',
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem',
      },
    },
  },
}
```

❌ 하드코딩된 스타일
```tsx
<div style={{ color: '#6366f1', padding: '16px' }}>
```

✅ 토큰 기반 스타일
```tsx
<div className="text-primary p-4">
```

## 실전 팁
- Storybook으로 컴포넌트 카탈로그 관리
- `cn()` 유틸로 조건부 클래스 안전하게 병합
- Figma Variables → CSS Variables로 자동화 검토
- 새 컴포넌트 추가 전 shadcn/ui에 있는지 먼저 확인

## 주의사항
- shadcn/ui 컴포넌트를 직접 수정할 때 업데이트 어려워짐 — 래핑 권장
- Tailwind arbitrary value (`text-[13px]`) 남용 — 토큰 추가가 우선
- 디자이너와 토큰 이름 합의 필수 — 코드와 Figma 불일치는 혼란 유발
