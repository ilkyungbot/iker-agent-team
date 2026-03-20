# Libraries — 라이브러리 선택 기준

> 라이브러리는 부채다. 번들 크기, 유지보수, 학습 비용을 감수할 가치가 있는지 먼저 따져라.

## 핵심 원칙
- **네이티브 우선**: Web API, React 내장으로 해결 가능하면 라이브러리 불필요
- **번들 크기 확인**: bundlephobia.com에서 크기 확인 후 채택
- **생태계 건강성**: weekly downloads, GitHub stars, 최근 커밋 확인
- **트리쉐이킹**: ESM 지원 여부 확인

## 승인된 라이브러리 목록
| 카테고리 | 라이브러리 | 비고 |
|----------|-----------|------|
| 상태 관리 | Zustand | Redux 대체 |
| 서버 상태 | TanStack Query | |
| 폼 | React Hook Form + Zod | |
| UI 컴포넌트 | shadcn/ui | |
| 날짜 | date-fns | moment.js 대체 |
| 애니메이션 | Framer Motion | |
| 테이블 | TanStack Table | |
| 차트 | Recharts | |
| 아이콘 | Lucide React | |
| 유틸 | es-toolkit | lodash 대체 |

## 코드 예시

✅ Zod로 런타임 타입 검증
```typescript
import { z } from 'zod'

const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.coerce.date(),
})

type User = z.infer<typeof UserSchema> // 타입 자동 생성

// API 응답 검증
async function fetchUser(id: string): Promise<User> {
  const data = await api.get(`/users/${id}`)
  return UserSchema.parse(data) // 실패 시 ZodError throw
}
```

✅ date-fns 사용
```typescript
import { format, formatDistance, addDays, isAfter } from 'date-fns'
import { ko } from 'date-fns/locale'

const formatted = format(new Date(), 'yyyy년 MM월 dd일', { locale: ko })
const relative = formatDistance(new Date(), new Date('2024-01-01'), { locale: ko })
```

✅ es-toolkit (lodash 대체)
```typescript
import { debounce, groupBy, chunk } from 'es-toolkit'

const debouncedSearch = debounce(handleSearch, 300)
const groupedItems = groupBy(items, (item) => item.category)
const pages = chunk(allItems, 10)
```

❌ 무거운 라이브러리 전체 import
```typescript
import _ from 'lodash' // 70KB+
import moment from 'moment' // 290KB+
```

✅ 경량 대안 또는 필요한 것만
```typescript
import { debounce } from 'es-toolkit' // 번들 최소화
import { format } from 'date-fns/format' // 서브모듈 import
```

## 실전 팁
- 새 라이브러리 채택 전 팀 리뷰 필수
- `peerDependencies` 충돌 확인 — React 버전 불일치 주의
- `package.json` `overrides`로 취약점 있는 하위 의존성 교체 가능

## 주의사항
- 라이브러리 1개 = 기여자, 보안 패치, 업그레이드 비용
- `npm install xxx` 전에 정말 필요한지 10초 생각
- 버려진 라이브러리 (`last commit > 2년`) 는 포크하거나 직접 구현 고려
