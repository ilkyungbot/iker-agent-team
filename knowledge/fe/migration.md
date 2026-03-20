# Migration — 기술 마이그레이션 전략

> 마이그레이션은 한 번에 다 바꾸는 것이 아니다. 점진적으로, 항상 동작하면서 이동하라.

## 핵심 원칙
- **점진적 마이그레이션**: 빅뱅보다 단계적 전환
- **Strangler Fig 패턴**: 새 코드가 구 코드를 서서히 대체
- **피처 플래그**: 새 구현과 구 구현을 동시에 운영
- **되돌리기 가능**: 각 단계는 롤백 가능해야 함

## 일반적인 마이그레이션 시나리오
| 시나리오 | 전략 |
|----------|------|
| JS → TypeScript | 파일 단위로 `.ts` 변환 |
| CRA → Next.js | 페이지 단위로 이전 |
| Redux → Zustand | store 단위로 이전 |
| REST → GraphQL | 엔드포인트 단위로 |
| CSS → Tailwind | 컴포넌트 단위로 |

## 코드 예시

✅ JavaScript → TypeScript 단계적 전환
```json
// tsconfig.json — 초기에는 관대하게 시작
{
  "compilerOptions": {
    "allowJs": true,          // JS 파일 허용
    "checkJs": false,         // JS 파일 타입 검사 끄기
    "strict": false,          // 처음엔 strict 끄기
    "noImplicitAny": false
  }
}
```

```typescript
// 마이그레이션 완료 후
{
  "compilerOptions": {
    "allowJs": false,
    "strict": true,
    "noImplicitAny": true
  }
}
```

✅ 피처 플래그로 안전한 전환
```typescript
// shared/lib/featureFlags.ts
const flags = {
  newCheckoutFlow: process.env.NEXT_PUBLIC_FF_NEW_CHECKOUT === 'true',
  newDashboard: process.env.NEXT_PUBLIC_FF_NEW_DASHBOARD === 'true',
} as const

export function useFeatureFlag(flag: keyof typeof flags): boolean {
  return flags[flag]
}

// 컴포넌트에서
function CheckoutPage() {
  const useNewFlow = useFeatureFlag('newCheckoutFlow')
  return useNewFlow ? <NewCheckout /> : <OldCheckout />
}
```

✅ 레거시 코드와 공존 (어댑터 패턴)
```typescript
// 구 API 클라이언트를 새 인터페이스로 감싸기
interface UserRepository {
  getById(id: string): Promise<User>
  update(id: string, data: Partial<User>): Promise<User>
}

class LegacyUserAdapter implements UserRepository {
  async getById(id: string): Promise<User> {
    const response = await legacyApi.fetchUser(id) // 구 API
    return this.mapToUser(response)                 // 새 타입으로 변환
  }

  private mapToUser(legacy: LegacyUser): User {
    return {
      id: legacy.user_id,        // snake_case → camelCase
      name: legacy.full_name,
      email: legacy.email_addr,
    }
  }
}
```

✅ 마이그레이션 체크리스트
```markdown
## CRA → Next.js 마이그레이션 단계
- [ ] 1단계: Next.js 설치, 기존 빌드와 병행
- [ ] 2단계: 라우팅 이전 (react-router → app router)
- [ ] 3단계: 데이터 페칭 이전 (useEffect → Server Component)
- [ ] 4단계: 성능 최적화 (next/image, next/font)
- [ ] 5단계: CRA 제거
```

## 실전 팁
- 마이그레이션 전 테스트 커버리지 확보 — 안전망 필수
- 대규모 마이그레이션은 전담 브랜치 대신 main에 점진적으로 머지
- 마이그레이션 진행률 Dashboard 만들어 팀과 공유

## 주의사항
- "일단 다 바꾸고 보자" 빅뱅 접근 — 롤백 불가, 버그 추적 어려움
- 마이그레이션 중 새 기능 추가 금지 — 혼란 가중
- 예상 기간 3배로 잡기 — 예외 처리와 엣지 케이스가 시간을 잡아먹음
