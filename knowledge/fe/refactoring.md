# Refactoring — 리팩터링 전략

> 리팩터링은 외부 동작을 유지하면서 내부 구조를 개선하는 것이다. 기능 추가와 동시에 하지 마라.

## 핵심 원칙
- **테스트 먼저**: 리팩터링 전 테스트로 안전망 구축
- **작은 단계**: 한 번에 하나씩, 항상 동작하는 상태 유지
- **보이스카우트 규칙**: 코드를 발견했을 때보다 나은 상태로 두고 가라
- **리팩터링 ≠ 기능 추가**: 두 가지 동시에 하지 말 것

## 리팩터링 신호 (Code Smells)
| 냄새 | 설명 | 해결 |
|------|------|------|
| Long Method | 함수 50줄 이상 | 함수 추출 |
| God Component | 컴포넌트 500줄 | 컴포넌트 분리 |
| Prop Drilling | props 3단계+ | Context/상태 끌어올리기 |
| Duplicate Code | 같은 로직 반복 | 훅/유틸 추출 |
| Dead Code | 사용 안 되는 코드 | 삭제 |

## 코드 예시

✅ 함수 추출 (Extract Function)
```typescript
// Before: 검증 로직이 제출 핸들러에 묻혀있음
function handleSubmit(data: FormData) {
  if (!data.email.includes('@')) throw new Error('Invalid email')
  if (data.password.length < 8) throw new Error('Password too short')
  if (data.password !== data.confirmPassword) throw new Error('Passwords do not match')
  api.signup(data)
}

// After: 검증 로직 분리
function validateSignupForm(data: FormData): void {
  if (!isValidEmail(data.email)) throw new Error('Invalid email')
  if (!isValidPassword(data.password)) throw new Error('Password too short')
  if (!isPasswordMatch(data.password, data.confirmPassword)) throw new Error('Passwords do not match')
}

function handleSubmit(data: FormData) {
  validateSignupForm(data)
  api.signup(data)
}
```

✅ 컴포넌트 분리
```typescript
// Before: 하나의 컴포넌트에 너무 많은 책임
function UserPage() {
  // 유저 데이터 로딩
  // 필터 상태 관리
  // 테이블 렌더링
  // 페이지네이션
  // 모달 처리
}

// After: 책임 분리
function UserPage() {
  return (
    <UserPageLayout>
      <UserFilters />
      <UserTable />
      <UserPagination />
    </UserPageLayout>
  )
}
```

✅ 커스텀 훅 추출
```typescript
// Before: 로직이 컴포넌트에 직접
function ProductList() {
  const [products, setProducts] = useState([])
  const [loading, setLoading] = useState(false)
  const [page, setPage] = useState(1)
  const [filters, setFilters] = useState({})

  useEffect(() => {
    setLoading(true)
    fetchProducts({ page, ...filters })
      .then(setProducts)
      .finally(() => setLoading(false))
  }, [page, filters])
  // ...
}

// After: 훅으로 추출
function useProductList() {
  const [page, setPage] = useState(1)
  const [filters, setFilters] = useState({})
  const { data: products, isLoading } = useQuery({
    queryKey: ['products', page, filters],
    queryFn: () => fetchProducts({ page, ...filters }),
  })
  return { products, isLoading, page, setPage, filters, setFilters }
}
```

## 실전 팁
- 리팩터링 PR은 기능 PR과 분리 — 리뷰 집중도 향상
- IDE 자동 리팩터링 기능 활용 (VS Code: F2 rename, Ctrl+Shift+R)
- `git stash` 활용 — 리팩터링 중 급한 버그 대응 시

## 주의사항
- "나중에 리팩터링" = 안 한다. 지금 할 수 없으면 TODO 티켓 생성
- 성능 최적화와 리팩터링은 별개 — 측정 없이 최적화 이름으로 구조 변경 금지
- 외부 API/인터페이스는 리팩터링 범위에서 신중하게 — 하위 호환성 고려
