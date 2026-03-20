# Data Fetching — 데이터 페칭 전략

> 언제, 어디서, 어떻게 데이터를 가져올지가 성능과 UX를 결정한다.

## 핵심 원칙
- **서버 컴포넌트 우선**: Next.js에서 가능하면 서버에서 fetch
- **폭포수 방지**: 병렬 요청, Promise.all 활용
- **캐싱 전략**: 데이터 특성에 맞는 캐시 정책
- **로딩 UI**: Skeleton, Suspense로 사용자 경험 보장

## 페칭 방법 선택 기준
| 상황 | 방법 |
|------|------|
| 서버에서 초기 데이터 | Server Component + fetch |
| 클라이언트 상호작용 | TanStack Query |
| 실시간 데이터 | Server-Sent Events / WebSocket |
| 정적 데이터 | generateStaticParams + SSG |

## 코드 예시

✅ Server Component에서 fetch
```typescript
// app/users/page.tsx (서버 컴포넌트)
async function UsersPage() {
  const [users, stats] = await Promise.all([
    fetch('/api/users', { next: { revalidate: 60 } }).then(r => r.json()),
    fetch('/api/stats', { cache: 'no-store' }).then(r => r.json()),
  ])

  return (
    <Suspense fallback={<UserListSkeleton />}>
      <UserList users={users} stats={stats} />
    </Suspense>
  )
}
```

✅ TanStack Query - Prefetch (서버 → 클라이언트 하이드레이션)
```typescript
// app/users/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'

export default async function UsersPage() {
  const queryClient = new QueryClient()
  await queryClient.prefetchQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserList /> {/* 클라이언트에서 캐시 재사용 */}
    </HydrationBoundary>
  )
}
```

✅ 폭포수 방지 — 병렬 쿼리
```typescript
// ❌ 폭포수: user 로드 후 posts 로드
function UserPage({ userId }: { userId: string }) {
  const { data: user } = useUser(userId)
  const { data: posts } = usePosts(user?.id) // user 로드 완료 후 시작
}

// ✅ 병렬 로드
function UserPage({ userId }: { userId: string }) {
  const [userQuery, postsQuery] = useQueries({
    queries: [
      { queryKey: ['user', userId], queryFn: () => fetchUser(userId) },
      { queryKey: ['posts', userId], queryFn: () => fetchPosts(userId) },
    ],
  })
}
```

✅ Skeleton 로딩 UI
```typescript
function UserCard() {
  const { data, isLoading } = useUser()

  if (isLoading) return <UserCardSkeleton />

  return <div>{data.name}</div>
}

function UserCardSkeleton() {
  return (
    <div className="animate-pulse">
      <div className="h-4 w-32 bg-muted rounded" />
      <div className="h-3 w-24 bg-muted rounded mt-2" />
    </div>
  )
}
```

## 실전 팁
- `staleTime` 설정으로 불필요한 재요청 방지
- `select` 옵션으로 필요한 데이터만 구독 (리렌더 최소화)
- `enabled` 조건으로 의존적 쿼리 제어

## 주의사항
- N+1 문제: 목록에서 각 항목마다 별도 요청 → 배치 API 설계 필요
- 클라이언트에서 secrets 노출 금지 — 서버 컴포넌트에서 처리
- `cache: 'no-store'` 남용 시 CDN 캐시 무력화
