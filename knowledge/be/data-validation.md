# Data Validation

## Zod 스키마 전략

```typescript
import { z } from 'zod'

// 기본 타입 정의
const emailSchema = z.string().email().toLowerCase().trim()
const uuidSchema  = z.string().uuid()
const nameSchema  = z.string().min(1).max(100).trim()

// 공통 페이지네이션
const paginationSchema = z.object({
  cursor: z.string().optional(),
  limit:  z.coerce.number().int().min(1).max(100).default(20),
  order:  z.enum(['asc', 'desc']).default('desc')
})

// 사용자 스키마 (계층 구조)
const createUserSchema = z.object({
  email:   emailSchema,
  name:    nameSchema,
  role:    z.enum(['admin', 'member']).default('member'),
  phone:   z.string().regex(/^01[0-9]-\d{3,4}-\d{4}$/).optional(),
  profile: z.object({
    bio:    z.string().max(500).optional(),
    avatar: z.string().url().optional()
  }).optional()
})

// 부분 업데이트 (모든 필드 선택적)
const updateUserSchema = createUserSchema.partial().omit({ email: true })

// 타입 추출
type CreateUserDTO = z.infer<typeof createUserSchema>
type UpdateUserDTO = z.infer<typeof updateUserSchema>
```

## 커스텀 검증

```typescript
// 비밀번호 강도
const passwordSchema = z.string()
  .min(8)
  .max(128)
  .refine(
    (pwd) => /[A-Z]/.test(pwd) && /[0-9]/.test(pwd) && /[!@#$%]/.test(pwd),
    { message: '대문자, 숫자, 특수문자(!@#$%) 포함 필요' }
  )

// 날짜 범위
const dateRangeSchema = z.object({
  startDate: z.coerce.date(),
  endDate:   z.coerce.date()
}).refine(
  (data) => data.endDate > data.startDate,
  { message: 'endDate must be after startDate', path: ['endDate'] }
)

// 한국 전화번호
const koreanPhoneSchema = z.string().regex(
  /^01[0-9]-\d{3,4}-\d{4}$/,
  '올바른 전화번호 형식 (010-1234-5678)'
)
```

## Fastify 통합

```typescript
import { zodToJsonSchema } from 'zod-to-json-schema'

// 자동 JSON Schema 변환
app.post('/users', {
  schema: {
    body:   zodToJsonSchema(createUserSchema),
    params: zodToJsonSchema(z.object({ id: uuidSchema }))
  },
  handler: async (req, reply) => {
    // 타입 안전하게 파싱
    const body = createUserSchema.parse(req.body)
    // ...
  }
})
```

## 입력 새니타이징

```typescript
// 문자열 정규화
const sanitizeString = (input: string) =>
  input
    .trim()
    .replace(/\s+/g, ' ')          // 연속 공백 제거
    .slice(0, 1000)                 // 길이 제한

// HTML 제거 (검색어 등)
const sanitizeSearchQuery = (query: string) =>
  query
    .replace(/<[^>]*>/g, '')        // HTML 태그 제거
    .replace(/[^\w\s가-힣]/g, ' ')  // 특수문자 제거
    .trim()
    .slice(0, 100)

// SQL 인젝션 방지 (ORM 사용이 기본이지만 추가 방어)
const preventSQLInjection = z.string().refine(
  (s) => !/(union|select|insert|update|delete|drop|create|alter|exec)/i.test(s),
  { message: 'Invalid characters in input' }
)
```

## 파일 업로드 검증

```typescript
const ALLOWED_MIME_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp'])
const MAX_FILE_SIZE      = 5 * 1024 * 1024  // 5MB

async function validateUploadedFile(file: MultipartFile) {
  if (!ALLOWED_MIME_TYPES.has(file.mimetype)) {
    throw new ValidationError({ file: 'Only JPEG, PNG, WebP allowed' })
  }

  const buffer = await file.toBuffer()
  if (buffer.length > MAX_FILE_SIZE) {
    throw new ValidationError({ file: 'File size exceeds 5MB' })
  }

  // Magic bytes 검증 (MIME 스푸핑 방지)
  const { fileTypeFromBuffer } = await import('file-type')
  const type = await fileTypeFromBuffer(buffer)
  if (!type || !ALLOWED_MIME_TYPES.has(type.mime)) {
    throw new ValidationError({ file: 'File content does not match declared type' })
  }

  return buffer
}
```

## 체크리스트
- [ ] 모든 외부 입력에 Zod 스키마 적용
- [ ] 문자열 trim + 길이 제한
- [ ] 파일 업로드 MIME 타입 + magic bytes 검증
- [ ] 에러 메시지에 민감 정보 미포함
- [ ] 검색어 새니타이징 (특수문자 처리)
