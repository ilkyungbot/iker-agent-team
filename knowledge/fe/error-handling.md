# Error Handling — 에러 처리 전략

> 에러는 발생한다. 언제 발생할지 모르는 에러보다, 어떻게 처리할지 모르는 것이 더 위험하다.

## 핵심 원칙
- **에러 분류**: 예상된 에러(비즈니스) vs 예상 못한 에러(버그)
- **사용자 친화적**: 기술적 메시지 대신 다음 행동 안내
- **에러 경계**: 컴포넌트 트리의 에러를 격리
- **로깅**: 에러는 Sentry 등으로 수집, 묻지 말 것

## 에러 레이어
| 레이어 | 처리 방법 |
|--------|----------|
| API/네트워크 | TanStack Query `onError` + toast |
| 폼 검증 | React Hook Form `setError` |
| 예상 못한 렌더 에러 | ErrorBoundary |
| 전역 처리 | `window.onerror` + Sentry |

## 코드 예시

✅ ErrorBoundary 구현
```typescript
'use client'
import { Component, type ReactNode } from 'react'

interface Props {
  fallback: ReactNode | ((error: Error) => ReactNode)
  children: ReactNode
}

interface State {
  error: Error | null
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null }

  static getDerivedStateFromError(error: Error): State {
    return { error }
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Sentry.captureException(error, { extra: info })
    console.error('ErrorBoundary caught:', error, info)
  }

  render() {
    if (this.state.error) {
      const { fallback } = this.props
      return typeof fallback === 'function' ? fallback(this.state.error) : fallback
    }
    return this.props.children
  }
}

// 사용
<ErrorBoundary
  fallback={(error) => (
    <div>
      <h2>오류가 발생했습니다</h2>
      <p>잠시 후 다시 시도해주세요</p>
      <button onClick={() => window.location.reload()}>새로고침</button>
    </div>
  )}
>
  <UserDashboard />
</ErrorBoundary>
```

✅ API 에러 타입 정의 및 처리
```typescript
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError
}

// 컴포넌트에서
const { mutate, error } = useMutation({ mutationFn: createUser })

if (isApiError(error)) {
  if (error.statusCode === 409) {
    return <Alert>이미 존재하는 사용자입니다</Alert>
  }
  if (error.statusCode === 422) {
    return <Alert>입력값을 확인해주세요</Alert>
  }
}
```

✅ Next.js error.tsx
```typescript
// app/dashboard/error.tsx
'use client'

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Sentry.captureException(error)
  }, [error])

  return (
    <div role="alert">
      <h2>대시보드를 불러올 수 없습니다</h2>
      <button onClick={reset}>다시 시도</button>
    </div>
  )
}
```

## 실전 팁
- Sentry `captureException`에 `user`, `tags` 컨텍스트 추가
- 에러 메시지 i18n: 코드 기반으로 관리 (`ERROR_CODES` 맵)
- React 19의 `useActionState`로 서버 액션 에러 처리

## 주의사항
- `try/catch`에서 에러 삼키기 금지 — 최소한 로깅
- 에러 상태에서 무한 루프 방지 — retry 횟수 제한
- 개발/프로덕션 에러 메시지 분리 — 스택 트레이스 노출 금지
