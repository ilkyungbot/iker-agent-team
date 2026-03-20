# Architecture — 프론트엔드 아키텍처 설계

> 좋은 아키텍처는 변경 비용을 최소화한다. 기술 결정보다 의존성 방향이 더 중요하다.

## 핵심 원칙
- **단방향 의존성**: UI → 비즈니스 로직 → 데이터 계층. 역방향 금지
- **관심사 분리**: 렌더링, 상태, 사이드이펙트를 명확히 구분
- **경계 명확화**: feature 간 직접 import 금지, shared 계층을 통해서만 교류
- **점진적 복잡도**: 필요해질 때 추상화하라. 미래를 위한 설계는 과설계다

## 디렉토리 구조 (Feature-Sliced 변형)
```
src/
  app/           # Next.js App Router (라우팅, 레이아웃)
  features/      # 독립 기능 단위 (auth, billing, dashboard)
    auth/
      components/
      hooks/
      store/
      api/
      types/
      index.ts   # public API만 export
  shared/        # 공통 유틸, UI 컴포넌트, 타입
    ui/
    lib/
    types/
  entities/      # 도메인 모델 (User, Order, Product)
```

## 판단 기준
| 질문 | Yes | No |
|------|-----|-----|
| 이 코드가 다른 feature에서도 쓰이나? | shared/ | features/내부 |
| 비즈니스 도메인 개념인가? | entities/ | features/ |
| 라우팅 설정인가? | app/ | - |

## 코드 예시

❌ feature 간 직접 의존
```typescript
// features/billing/components/BillingForm.tsx
import { useAuthStore } from '../auth/store/authStore' // 금지!
```

✅ shared 계층을 통한 교류
```typescript
// features/billing/components/BillingForm.tsx
import { useCurrentUser } from '@/shared/hooks/useCurrentUser'
```

❌ 레이어 역전 (데이터 계층이 UI를 알면 안 됨)
```typescript
// api/userApi.ts
import { UserCard } from '@/features/user/components/UserCard' // 금지!
```

✅ 올바른 의존성 방향
```typescript
// features/user/components/UserCard.tsx
import { fetchUser } from '@/features/user/api/userApi'
```

## 실전 팁
- `index.ts` 배럴 파일로 내부 구현 숨기기. 외부에선 `features/auth`만 import
- 새 feature 시작 시 `types/index.ts`부터 작성 — 인터페이스가 설계를 강제함
- 순환 의존성 감지: `madge --circular src/` 를 CI에 추가
- 서버 컴포넌트 vs 클라이언트 컴포넌트 경계를 아키텍처 레벨에서 결정

## 주의사항
- God Component: 500줄 이상 컴포넌트는 무조건 분리 신호
- Prop Drilling 3단계 이상 → Context 또는 상태 끌어올리기 고려
- "나중에 리팩터링하자" = 영원히 안 한다. 지금 올바른 위치에 두어라
