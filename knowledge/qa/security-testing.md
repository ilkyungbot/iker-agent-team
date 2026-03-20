# 보안 테스트 — 공격자의 시선으로 코드를 바라본다

> 보안 취약점은 발견 시점이 늦을수록 수정 비용이 100배 이상 증가한다.

## 핵심 원칙
- Shift Left: 개발 초기부터 보안을 통합한다
- OWASP Top 10을 체크리스트로 활용한다
- 자동화(SAST/DAST)와 수동 검토를 병행한다
- 인증/인가 로직은 화이트박스와 블랙박스 모두 테스트한다

## 판단 기준
필수 보안 테스트 영역:
- **인증**: JWT 위변조, 만료 토큰 재사용, 세션 고정
- **인가**: 수평적 권한 상승, 수직적 권한 상승, IDOR
- **입력 검증**: SQL 인젝션, XSS, 명령어 인젝션
- **민감 데이터**: 응답에 패스워드/키 노출, 로그에 PII 기록

## 코드 예시 (TypeScript/Vitest)
```typescript
// src/security/auth.security.test.ts
describe('인가 보안 테스트', () => {
  it('다른 사용자의 리소스에 접근할 수 없다 (IDOR 방지)', async () => {
    const userA = await createTestUser()
    const userB = await createTestUser()
    const userADoc = await createDocument({ ownerId: userA.id })

    // userB가 userA의 문서에 접근 시도
    const response = await request(app)
      .get(`/api/documents/${userADoc.id}`)
      .set('Authorization', `Bearer ${userB.token}`)

    expect(response.status).toBe(403)
  })

  it('만료된 JWT로 보호된 엔드포인트에 접근할 수 없다', async () => {
    const expiredToken = generateJWT({ userId: '123', expiresIn: '-1s' })

    const response = await request(app)
      .get('/api/profile')
      .set('Authorization', `Bearer ${expiredToken}`)

    expect(response.status).toBe(401)
    expect(response.body.message).toBe('토큰이 만료되었습니다')
  })

  it('SQL 인젝션 시도를 차단한다', async () => {
    const maliciousInput = "'; DROP TABLE users; --"
    const response = await request(app)
      .get(`/api/users?search=${encodeURIComponent(maliciousInput)}`)
      .set('Authorization', `Bearer ${adminToken}`)

    expect(response.status).toBe(200)
    // DB가 손상되지 않았음을 확인
    const users = await userRepo.findAll()
    expect(users.length).toBeGreaterThan(0)
  })
})
```

## 안티패턴
- 보안 테스트를 별도 보안팀에만 위임
- Happy Path만 테스트하고 악의적 입력 미검증
- 에러 응답에 스택 트레이스/DB 스키마 노출
- `console.log`에 토큰, 패스워드, PII 출력

## 실전 팁
- GitHub Actions에 `npm audit`, `snyk test`를 자동 연동한다
- `.env` 파일을 절대 커밋하지 않도록 `.gitignore` + pre-commit hook을 설정한다
- 보안 헤더(`Content-Security-Policy`, `X-Frame-Options`)는 통합 테스트에서 검증한다
- 의존성 취약점 스캔을 매주 Dependabot으로 자동화한다
