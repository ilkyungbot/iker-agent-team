# Routing — Next.js App Router 라우팅

> URL은 사용자와의 계약이다. 변경하면 사용자가 저장한 링크가 깨진다.

## 핵심 원칙
- **URL = 상태**: 필터, 검색어, 페이지는 URL에 저장
- **레이아웃 공유**: 중첩 라우팅으로 공통 UI 재사용
- **병렬 라우트**: 독립적인 섹션은 병렬 라우트로
- **서버/클라이언트 경계**: 라우트 단위에서 결정

## App Router 디렉토리 구조
```
app/
  (auth)/             # 라우트 그룹 (URL에 영향 없음)
    login/page.tsx
    signup/page.tsx
  (dashboard)/
    layout.tsx         # 대시보드 공통 레이아웃
    page.tsx           # /
    users/
      page.tsx         # /users
      [id]/
        page.tsx       # /users/[id]
        edit/page.tsx  # /users/[id]/edit
  @modal/             # 병렬 라우트
    (.)users/[id]/page.tsx  # 인터셉트 라우트
  loading.tsx          # Suspense fallback
  error.tsx            # ErrorBoundary
  not-found.tsx
```

## 코드 예시

✅ URL 상태 관리
```typescript
'use client'
import { useRouter, useSearchParams, usePathname } from 'next/navigation'

function UserFilter() {
  const router = useRouter()
  const pathname = usePathname()
  const searchParams = useSearchParams()

  function setFilter(key: string, value: string) {
    const params = new URLSearchParams(searchParams.toString())
    params.set(key, value)
    router.push(`${pathname}?${params.toString()}`)
  }

  const role = searchParams.get('role') ?? 'all'
  return (
    <select value={role} onChange={(e) => setFilter('role', e.target.value)}>
      <option value="all">전체</option>
      <option value="admin">관리자</option>
    </select>
  )
}
```

✅ 동적 라우트 + generateStaticParams
```typescript
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getAllPosts()
  return posts.map((post) => ({ slug: post.slug }))
}

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)
  if (!post) notFound()
  return <article>{post.content}</article>
}
```

✅ Parallel Routes로 모달
```typescript
// app/layout.tsx
export default function Layout({
  children,
  modal,
}: {
  children: React.ReactNode
  modal: React.ReactNode
}) {
  return (
    <>
      {children}
      {modal}
    </>
  )
}
```

✅ Middleware로 인증 처리
```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')
  const isAuthPage = request.nextUrl.pathname.startsWith('/login')

  if (!token && !isAuthPage) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

## 실전 팁
- `useTransition` + `router.push`로 라우트 전환 시 pending 상태 표시
- `prefetch={false}`로 불필요한 프리페치 방지
- Breadcrumb은 usePathname()으로 동적 생성

## 주의사항
- Pages Router와 App Router 혼용 시 충돌 가능성
- `searchParams`는 서버 컴포넌트에서 prop으로, 클라이언트에서 훅으로
- `router.push` vs `router.replace` — 뒤로가기 필요 여부로 선택
