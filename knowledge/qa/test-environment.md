# 테스트 환경 — 일관된 환경이 신뢰할 수 있는 테스트를 만든다

> 테스트가 "내 로컬에서는 됨"이면 환경 문제다. 환경을 코드로 관리하라.

## 핵심 원칙
- 환경은 코드로 정의하고 버전 관리한다 (Docker, IaC)
- 개발/테스트/스테이징/프로덕션 환경은 구조가 같아야 한다
- 환경 변수는 `.env.example`로 문서화하고 실제 값은 시크릿으로 관리한다
- 테스트 환경은 테스트 후 원래 상태로 복원된다

## 판단 기준
환경 구성 전략:
| 환경 | DB | 외부 API | 이메일 |
|------|-----|---------|--------|
| 로컬 단위 테스트 | 모킹 | MSW | 모킹 |
| 로컬 통합 테스트 | Docker | MSW | MailHog |
| CI | Docker | MSW | 모킹 |
| 스테이징 | 전용 DB | Sandbox | Mailtrap |
| 프로덕션 | 프로덕션 DB | 실제 | 실제 |

## 코드 예시 (Docker Compose + TypeScript)
```yaml
# docker-compose.test.yml
version: '3.9'
services:
  postgres-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    ports:
      - '5433:5432'
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'testuser']
      interval: 5s
      retries: 5

  redis-test:
    image: redis:7-alpine
    ports:
      - '6380:6379'

  mailhog:
    image: mailhog/mailhog
    ports:
      - '8025:8025'  # Web UI
      - '1025:1025'  # SMTP
```

```typescript
// vitest.globalSetup.ts
import { exec } from 'node:child_process'
import { promisify } from 'node:util'

const execAsync = promisify(exec)

export async function setup() {
  await execAsync('docker-compose -f docker-compose.test.yml up -d --wait')
  process.env.DATABASE_URL = 'postgresql://testuser:testpass@localhost:5433/testdb'
  process.env.REDIS_URL = 'redis://localhost:6380'
  // 마이그레이션 실행
  await execAsync('npx prisma migrate deploy')
}

export async function teardown() {
  await execAsync('docker-compose -f docker-compose.test.yml down -v')
}
```

## 안티패턴
- 로컬 머신에만 설치된 서비스에 의존
- 공유 테스트 DB에서 병렬 테스트 실행
- 환경 변수를 하드코딩 (다른 환경에서 실패)
- 테스트 환경이 프로덕션과 버전 차이가 큰 경우

## 실전 팁
- `vitest`의 `globalSetup`/`globalTeardown`으로 컨테이너 생명주기를 관리한다
- `.env.test` 파일로 테스트 전용 환경 변수를 분리한다
- Testcontainers 라이브러리로 코드에서 직접 컨테이너를 제어한다
- CI 캐싱으로 Docker 이미지 풀 시간을 최소화한다
