# Database Migration

## Drizzle Kit 설정

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema:    './src/db/schema.ts',
  out:       './drizzle/migrations',
  dialect:   'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL! },
  verbose:   true,
  strict:    true
})
```

```bash
# 마이그레이션 파일 생성 (스키마 변경 후)
npx drizzle-kit generate

# 마이그레이션 실행
npx drizzle-kit migrate

# 현재 상태 확인
npx drizzle-kit studio
```

## 마이그레이션 전략

```sql
-- 안전한 컬럼 추가 (Backward Compatible)
ALTER TABLE users ADD COLUMN phone text;           -- NULL 허용으로 시작
ALTER TABLE users ADD COLUMN phone text NOT NULL DEFAULT '';  -- 기본값 제공

-- 위험한 작업 (영향 분석 필요)
-- ALTER TABLE users DROP COLUMN old_field;
-- ALTER TABLE users ALTER COLUMN status TYPE ...;
-- CREATE INDEX ON large_table ...;  -- CONCURRENTLY 사용 필수
```

## Zero-Downtime 마이그레이션 원칙

```sql
-- 1단계: 새 컬럼 추가 (애플리케이션 배포 전)
ALTER TABLE orders ADD COLUMN new_status text;

-- 2단계: 코드 양쪽 모두 지원하도록 업데이트
-- (new_status 없으면 status 폴백)

-- 3단계: 데이터 백필 (배치로)
UPDATE orders SET new_status = status WHERE new_status IS NULL LIMIT 1000;

-- 4단계: 제약 조건 추가
ALTER TABLE orders ALTER COLUMN new_status SET NOT NULL;

-- 5단계: 구 컬럼 제거 (다음 배포 사이클)
ALTER TABLE orders DROP COLUMN status;
```

## 대용량 테이블 마이그레이션

```sql
-- 인덱스: CONCURRENTLY로 락 없이 생성
CREATE INDEX CONCURRENTLY users_email_idx ON users (email);

-- 컬럼 추가: 기본값 있는 경우 즉시 완료 (PostgreSQL 11+)
ALTER TABLE large_table ADD COLUMN flag boolean NOT NULL DEFAULT false;

-- 대량 UPDATE: 배치 처리
DO $$
DECLARE
  batch_size INT := 10000;
  offset_val INT := 0;
  updated    INT;
BEGIN
  LOOP
    UPDATE orders SET new_column = computed_value
    WHERE id IN (
      SELECT id FROM orders WHERE new_column IS NULL
      ORDER BY id LIMIT batch_size
    );
    GET DIAGNOSTICS updated = ROW_COUNT;
    EXIT WHEN updated = 0;
    PERFORM pg_sleep(0.1);  -- 잠시 대기 (부하 분산)
  END LOOP;
END $$;
```

## 롤백 전략

```typescript
// 마이그레이션 실행 with 롤백
async function runMigrations() {
  try {
    await migrate(db, { migrationsFolder: './drizzle/migrations' })
    logger.info('Migrations completed successfully')
  } catch (err) {
    logger.error({ err }, 'Migration failed — rolling back')
    // Drizzle은 트랜잭션 내에서 실행하므로 자동 롤백
    process.exit(1)
  }
}
```

## 체크리스트
- [ ] 마이그레이션 파일을 Git에 커밋
- [ ] 프로덕션 적용 전 스테이징 환경 테스트
- [ ] 대용량 테이블 DDL은 CONCURRENTLY 사용
- [ ] 배포 전 롤백 계획 수립
- [ ] 마이그레이션 실행 시간 측정 (다운타임 예측)
- [ ] 백업 후 마이그레이션 실행
