# Security — 프론트엔드 보안

> 프론트엔드 보안은 방어선이 아니라 방어의 일부다. 서버가 최종 방어선이다.

## 핵심 원칙
- **XSS 방지**: 사용자 입력을 절대 dangerouslySetInnerHTML에 직접 삽입 금지
- **CSRF 보호**: SameSite 쿠키, CSRF 토큰
- **secrets 노출 금지**: API 키, 토큰을 클라이언트 번들에 포함 금지
- **의존성 취약점**: `npm audit` 정기 실행

## OWASP Top 10 관련 항목
| 위협 | 대응 |
|------|------|
| XSS | React 자동 이스케이프, DOMPurify |
| CSRF | SameSite=Strict, CSRF 토큰 |
| 민감정보 노출 | 환경변수 관리, HTTPS 강제 |
| 취약한 의존성 | npm audit, Dependabot |

## 코드 예시

❌ XSS 취약점
```typescript
// 절대 금지
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// eval, new Function도 금지
eval(userCode)
```

✅ HTML 삽입이 필요할 때 (예: CMS 콘텐츠)
```typescript
import DOMPurify from 'dompurify'

function RichContent({ html }: { html: string }) {
  const sanitized = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'strong', 'em', 'a', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: ['href', 'target'],
  })
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />
}
```

✅ 환경변수 관리
```bash
# .env.local (git 제외)
DATABASE_URL=...          # 서버에서만 사용
NEXT_PUBLIC_API_URL=...   # 클라이언트에 노출 가능 (NEXT_PUBLIC_ 접두사)

# 절대 클라이언트에 노출 금지
SECRET_KEY=...
STRIPE_SECRET_KEY=...
```

```typescript
// 서버 컴포넌트 또는 API route에서만
const apiKey = process.env.STRIPE_SECRET_KEY // 클라이언트 번들에 포함 안 됨

// 클라이언트 컴포넌트에서 접근 가능
const publicUrl = process.env.NEXT_PUBLIC_API_URL
```

✅ CSP (Content Security Policy) 헤더
```javascript
// next.config.js
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'nonce-${nonce}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
`

module.exports = {
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        { key: 'Content-Security-Policy', value: ContentSecurityPolicy },
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
      ],
    },
  ],
}
```

✅ 인증 토큰 안전 저장
```typescript
// localStorage 대신 HttpOnly 쿠키 사용 (XSS로 탈취 불가)
// 서버에서 Set-Cookie: token=...; HttpOnly; Secure; SameSite=Strict

// 불가피하게 localStorage 사용 시
const tokenManager = {
  set: (token: string) => {
    sessionStorage.setItem('token', token) // 탭 닫으면 삭제
  },
  get: () => sessionStorage.getItem('token'),
  clear: () => sessionStorage.removeItem('token'),
}
```

## 실전 팁
- `npm audit --audit-level=high` CI 파이프라인에 추가
- Dependabot 또는 Renovate로 자동 보안 패치 PR
- `.env` 파일 절대 git commit 금지 — `.gitignore` 확인

## 주의사항
- "프론트에서 검증했으니 서버는 안 해도 됨" — 절대 금지
- URL 파라미터로 받은 HTML 직접 렌더링 금지
- third-party 스크립트는 신뢰할 수 있는 출처만
