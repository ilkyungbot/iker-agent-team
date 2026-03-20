# Code Quality

## TypeScript 엄격 설정

```json
// tsconfig.json
{
  "compilerOptions": {
    "target":                  "ES2022",
    "module":                  "NodeNext",
    "moduleResolution":        "NodeNext",
    "strict":                  true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride":      true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck":            true,
    "outDir":                  "dist"
  }
}
```

## ESLint 설정

```javascript
// eslint.config.mjs
import js       from '@eslint/js'
import ts       from '@typescript-eslint/eslint-plugin'
import tsParser from '@typescript-eslint/parser'

export default [
  js.configs.recommended,
  {
    files:   ['src/**/*.ts'],
    plugins: { '@typescript-eslint': ts },
    parser:  tsParser,
    rules: {
      '@typescript-eslint/no-explicit-any':        'error',
      '@typescript-eslint/no-unused-vars':          ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/explicit-function-return-type': 'warn',
      'no-console':                                  'warn',
      'prefer-const':                                'error',
      'no-var':                                      'error',
      'eqeqeq':                                      ['error', 'always']
    }
  }
]
```

## 함수 설계 원칙

```typescript
// 좋음: 단일 책임, 순수 함수
function calculateDiscount(price: number, rate: number): number {
  if (price < 0) throw new Error('Price must be non-negative')
  if (rate < 0 || rate > 1) throw new Error('Rate must be between 0 and 1')
  return Math.round(price * (1 - rate))
}

// 좋음: 이름이 명확한 함수
async function sendWelcomeEmailToNewUser(user: User): Promise<void> { ... }

// 나쁨: 모호한 이름, 여러 책임
async function process(u: any): Promise<any> { ... }

// 불변성 유지
function updateUser(user: User, changes: Partial<User>): User {
  return { ...user, ...changes, updatedAt: new Date() }  // 원본 수정 X
}
```

## 의존성 관리

```typescript
// 의존성 주입으로 테스트 가능성 확보
class OrderService {
  constructor(
    private readonly orderRepo:   OrderRepository,
    private readonly userService: UserService,
    private readonly emailQueue:  Queue,
    private readonly logger:      Logger
  ) {}
}

// 팩토리 함수 패턴
function createOrderService(container: Container): OrderService {
  return new OrderService(
    container.resolve(OrderRepository),
    container.resolve(UserService),
    container.resolve('emailQueue'),
    container.resolve(Logger)
  )
}
```

## 코드 리뷰 체크리스트

```markdown
## PR 작성자
- [ ] 변경 범위를 명확히 설명
- [ ] 테스트 케이스 포함
- [ ] 브레이킹 체인지 명시
- [ ] 성능 영향 분석

## 리뷰어
- [ ] 비즈니스 로직 정확성
- [ ] 에러 처리 완결성
- [ ] 보안 취약점 (인증/인가, SQL 인젝션)
- [ ] 성능 (N+1, 인덱스)
- [ ] 테스트 커버리지
```

## 코드 복잡도 기준

| 지표 | 권장 | 경고 |
|------|------|------|
| 함수 길이 | < 30줄 | > 50줄 |
| 순환 복잡도 | < 5 | > 10 |
| 파일 길이 | < 200줄 | > 400줄 |
| 파라미터 수 | < 4개 | > 6개 |

## 체크리스트
- [ ] `any` 타입 사용 금지
- [ ] 모든 공개 함수 반환 타입 명시
- [ ] 매직 넘버/문자열을 상수로 추출
- [ ] 함수 길이 30줄 이내
- [ ] Husky + lint-staged로 커밋 전 자동 검사
