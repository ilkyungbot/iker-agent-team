# Async Patterns — 비동기 패턴

> 비동기 코드의 복잡성은 상태 관리에서 온다. loading/error/data를 명확히 다루어라.

## 핵심 원칙
- **TanStack Query 우선**: 서버 상태의 fetching, caching, 동기화는 Query로
- **명시적 상태**: loading, error, success, idle 상태 모두 처리
- **낙관적 업데이트**: UX 향상을 위해 서버 응답 전에 UI 업데이트
- **에러 경계**: 예상 못한 에러는 ErrorBoundary가 처리

## 코드 예시

✅ TanStack Query 기본 패턴
```typescript
// query key factory
const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  detail: (id: string) => [...userKeys.all, 'detail', id] as const,
}

// 커스텀 훅으로 추상화
function useUsers() {
  return useQuery({
    queryKey: userKeys.lists(),
    queryFn: () => userApi.getAll(),
    staleTime: 60 * 1000, // 1분
  })
}

function useUser(id: string) {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => userApi.getById(id),
    enabled: !!id,
  })
}
```

✅ Mutation + 낙관적 업데이트
```typescript
function useUpdateUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (user: UpdateUserInput) => userApi.update(user),
    onMutate: async (newUser) => {
      await queryClient.cancelQueries({ queryKey: userKeys.detail(newUser.id) })
      const previous = queryClient.getQueryData(userKeys.detail(newUser.id))
      queryClient.setQueryData(userKeys.detail(newUser.id), newUser)
      return { previous }
    },
    onError: (err, newUser, context) => {
      queryClient.setQueryData(userKeys.detail(newUser.id), context?.previous)
    },
    onSettled: (_, __, newUser) => {
      queryClient.invalidateQueries({ queryKey: userKeys.detail(newUser.id) })
    },
  })
}
```

✅ 무한 스크롤
```typescript
function useInfiniteUsers() {
  return useInfiniteQuery({
    queryKey: userKeys.lists(),
    queryFn: ({ pageParam = 0 }) => userApi.getPaged({ page: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    initialPageParam: 0,
  })
}
```

✅ async/await 에러 처리
```typescript
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  retries = 3
): Promise<T> {
  try {
    return await fn()
  } catch (error) {
    if (retries > 0) return fetchWithRetry(fn, retries - 1)
    throw error
  }
}
```

## 실전 팁
- `suspense: true` 옵션으로 Suspense와 통합
- `prefetchQuery`로 라우트 전환 전 데이터 미리 로드
- Devtools: `@tanstack/react-query-devtools` 개발 중 항상 켜두기
- 에러는 `useErrorBoundary: true`로 ErrorBoundary에 위임 가능

## 주의사항
- useEffect로 데이터 페칭 금지 — TanStack Query 사용
- Promise.all은 하나라도 실패하면 전체 실패 — Promise.allSettled 검토
- 무한 쿼리에서 캐시 메모리 관리 주의
