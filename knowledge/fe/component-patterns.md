# Component Patterns — 컴포넌트 패턴

> 컴포넌트는 UI의 레고 블록이다. 작고 명확하고 조합 가능하게 만들어라.

## 핵심 원칙
- **단일 책임**: 컴포넌트 변경 이유가 2개 이상이면 분리
- **합성 우선**: 상속보다 합성, props drilling보다 children
- **제어 컴포넌트**: 상태를 외부에서 제어 가능하게
- **headless 패턴**: 로직과 스타일 분리

## 패턴 목록

### 1. Compound Component (복합 컴포넌트)
```typescript
// 사용 예시
<Tabs defaultValue="account">
  <Tabs.List>
    <Tabs.Trigger value="account">계정</Tabs.Trigger>
    <Tabs.Trigger value="security">보안</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="account"><AccountForm /></Tabs.Content>
  <Tabs.Content value="security"><SecurityForm /></Tabs.Content>
</Tabs>
```

### 2. Render Props / Children as Function
```typescript
interface DataTableProps<T> {
  data: T[]
  renderRow: (item: T, index: number) => React.ReactNode
}

function DataTable<T>({ data, renderRow }: DataTableProps<T>) {
  return (
    <table>
      <tbody>
        {data.map((item, i) => (
          <tr key={i}>{renderRow(item, i)}</tr>
        ))}
      </tbody>
    </table>
  )
}
```

### 3. 제어/비제어 컴포넌트 통합
```typescript
interface InputProps {
  value?: string           // 제어 모드
  defaultValue?: string    // 비제어 모드
  onChange?: (value: string) => void
}

function Input({ value, defaultValue, onChange }: InputProps) {
  const [internalValue, setInternalValue] = useState(defaultValue ?? '')
  const isControlled = value !== undefined
  const displayValue = isControlled ? value : internalValue

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (!isControlled) setInternalValue(e.target.value)
    onChange?.(e.target.value)
  }

  return <input value={displayValue} onChange={handleChange} />
}
```

### 4. HOC (Higher Order Component) — 신중하게 사용
```typescript
function withAuth<P extends object>(Component: React.ComponentType<P>) {
  return function AuthenticatedComponent(props: P) {
    const { isAuthenticated } = useAuthStore()
    if (!isAuthenticated) return <Navigate to="/login" />
    return <Component {...props} />
  }
}
```

### 5. 커스텀 훅으로 로직 분리
```typescript
// ❌ 로직이 컴포넌트에 섞여있음
function SearchPage() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const debouncedQuery = useDebounce(query, 300)
  useEffect(() => { /* 검색 로직 */ }, [debouncedQuery])
  // ...렌더링 코드
}

// ✅ 훅으로 로직 분리
function useSearch(initialQuery = '') {
  const [query, setQuery] = useState(initialQuery)
  const debouncedQuery = useDebounce(query, 300)
  const { data: results } = useQuery({ queryKey: ['search', debouncedQuery], queryFn: () => search(debouncedQuery) })
  return { query, setQuery, results }
}

function SearchPage() {
  const { query, setQuery, results } = useSearch()
  // 순수 렌더링 코드
}
```

## 실전 팁
- `forwardRef`로 ref 전달 지원 — UI 라이브러리 필수
- `displayName` 설정 — DevTools에서 컴포넌트 이름 표시
- `React.memo`는 props가 자주 바뀌지 않고 렌더 비용이 클 때만

## 주의사항
- HOC 중첩 3개 이상 → 가독성 급락, 커스텀 훅으로 대체
- Props 15개 이상 → 컴포넌트 분리 또는 설계 재검토 신호
