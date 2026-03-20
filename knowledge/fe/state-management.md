# State Management — 상태 관리 전략

> 상태는 가능한 한 가까이, 가능한 한 작게. 전역 상태는 정말 전역이어야 할 때만.

## 핵심 원칙
- **상태 최소화**: 파생 가능한 값은 상태가 아닌 계산값으로
- **상태 배치**: 사용하는 컴포넌트에 가장 가까운 곳에
- **서버 상태 분리**: 서버 데이터는 TanStack Query, 클라이언트 상태는 Zustand/useState
- **단방향 데이터 흐름**: 상태를 아래로, 이벤트를 위로

## 상태 종류별 도구 선택
| 상태 종류 | 도구 | 예시 |
|----------|------|------|
| 로컬 UI 상태 | useState | 모달 open/close |
| 복잡한 로컬 상태 | useReducer | 멀티스텝 폼 |
| 서버 데이터 | TanStack Query | 유저 목록 |
| 전역 클라이언트 | Zustand | 인증 정보, 테마 |
| URL 상태 | useSearchParams | 필터, 페이지 |
| 폼 상태 | React Hook Form | 입력 폼 |

## 코드 예시

❌ 파생 값을 상태로 관리
```typescript
const [items, setItems] = useState<Item[]>([])
const [total, setTotal] = useState(0) // 파생값을 상태로!

useEffect(() => {
  setTotal(items.reduce((sum, item) => sum + item.price, 0))
}, [items])
```

✅ 파생 값은 계산
```typescript
const [items, setItems] = useState<Item[]>([])
const total = items.reduce((sum, item) => sum + item.price, 0) // 렌더마다 계산
```

✅ Zustand store 설계
```typescript
interface AuthStore {
  user: User | null
  isAuthenticated: boolean // 파생값이지만 자주 쓰여 포함
  login: (credentials: Credentials) => Promise<void>
  logout: () => void
}

const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  isAuthenticated: false,
  login: async (credentials) => {
    const user = await authApi.login(credentials)
    set({ user, isAuthenticated: true })
  },
  logout: () => set({ user: null, isAuthenticated: false }),
}))
```

✅ TanStack Query + Zustand 조합
```typescript
// 서버 상태: TanStack Query
function useUserProfile(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => userApi.getProfile(userId),
    staleTime: 5 * 60 * 1000,
  })
}

// 클라이언트 상태: Zustand
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}))
```

## 실전 팁
- Zustand devtools 미들웨어로 상태 변화 추적
- `immer` 미들웨어로 불변성 관리 단순화
- TanStack Query `queryKey` 팩토리 함수로 키 일관성 유지
- 전역 상태 추가 전 URL 상태나 props로 해결 가능한지 먼저 확인

## 주의사항
- Context API로 자주 바뀌는 상태 관리 → 불필요한 리렌더 유발
- Redux를 굳이 추가하지 말 것 — Zustand + TanStack Query로 충분
- Zustand store를 너무 잘게 쪼개지 말 것 — 관련 상태는 같은 store에
