# Code Quality — 읽히는 코드 작성법

> 코드는 컴퓨터가 아닌 사람이 읽는다. 명확함이 영리함을 이긴다.

## 핵심 원칙
- **명명이 곧 문서**: 변수/함수 이름만으로 의도를 파악할 수 있어야 함
- **작은 함수**: 한 함수는 한 가지 일만. 20줄 초과 시 분리 고려
- **조기 반환**: 중첩 조건문 대신 guard clause 사용
- **`any` 금지**: 타입을 모르면 `unknown`으로 시작해 narrowing

## 판단 기준
- 함수명에 'and'가 들어가면 → 두 함수로 분리
- 주석으로 코드 블록을 설명하고 있다면 → 함수 추출 신호
- Boolean 파라미터 → 두 함수로 분리 또는 옵션 객체 사용

## 코드 예시

❌ 나쁜 명명 + 중첩 구조
```typescript
function process(d: any, f: boolean) {
  if (d) {
    if (f) {
      // 유저 처리
      d.name = d.name.trim()
      d.email = d.email.toLowerCase()
    }
  }
}
```

✅ 명확한 명명 + guard clause
```typescript
function normalizeUserInput(user: User): User {
  if (!user) return user

  return {
    ...user,
    name: user.name.trim(),
    email: user.email.toLowerCase(),
  }
}
```

❌ Boolean 파라미터 함정
```typescript
renderButton(true)  // true가 뭘 의미하는지 모름
```

✅ 명시적 옵션
```typescript
renderButton({ variant: 'primary', isLoading: true })
```

❌ 타입 any 남용
```typescript
function parseResponse(data: any) {
  return data.user.id // 런타임 에러 위험
}
```

✅ unknown + type narrowing
```typescript
function parseResponse(data: unknown): string {
  if (
    typeof data === 'object' &&
    data !== null &&
    'user' in data &&
    typeof (data as { user: unknown }).user === 'object'
  ) {
    const { user } = data as { user: { id: string } }
    return user.id
  }
  throw new Error('Invalid response shape')
}
```

## 실전 팁
- 함수 추출 전 테스트 먼저 작성 → 리팩터링 안전망 확보
- `TODO:` 주석에 날짜와 이름 추가: `// TODO(2024-03-15 @yourname): ...`
- ESLint `no-nested-ternary`, `@typescript-eslint/no-explicit-any` 활성화
- 리뷰 시 "이 코드를 6개월 뒤 내가 읽는다면?" 질문

## 주의사항
- 지나친 DRY: 비슷해 보이는 코드가 다른 변경 이유를 가진다면 중복이 더 낫다
- 과도한 추상화: 3번 이상 반복되기 전에 추상화하지 마라
- 스마트한 코드: 동료가 이해 못 하는 코드는 좋은 코드가 아님
