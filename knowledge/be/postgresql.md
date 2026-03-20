# PostgreSQL 16+ 고급 활용

## 연결 풀 설정 (pg + Drizzle)

```typescript
import { drizzle } from 'drizzle-orm/node-postgres'
import { Pool } from 'pg'

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max:              20,     // 최대 연결 수
  idleTimeoutMillis: 30000, // 유휴 연결 제거 대기
  connectionTimeoutMillis: 5000,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
})

pool.on('error', (err) => {
  logger.error({ err }, 'PostgreSQL pool error')
  process.exit(1)
})

export const db = drizzle(pool, { schema, logger: true })
```

## JSONB 컬럼 활용

```typescript
import { jsonb } from 'drizzle-orm/pg-core'

const products = pgTable('products', {
  id:       uuid('id').primaryKey().defaultRandom(),
  metadata: jsonb('metadata').$type<ProductMetadata>()
})

// JSONB 인덱스 (GIN)
// migration SQL:
// CREATE INDEX products_metadata_gin_idx ON products USING gin(metadata);

// JSONB 쿼리
await db.execute(sql`
  SELECT * FROM products
  WHERE metadata @> '{"category": "electronics"}'::jsonb
`)
```

## 전문 검색 (Full-text Search)

```typescript
// tsvector 컬럼 추가 (migration)
// ALTER TABLE products ADD COLUMN search_vector tsvector
//   GENERATED ALWAYS AS (
//     to_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,''))
//   ) STORED;
// CREATE INDEX products_search_idx ON products USING GIN(search_vector);

// 검색 쿼리
const results = await db.execute(sql`
  SELECT *, ts_rank(search_vector, query) AS rank
  FROM products, to_tsquery('english', ${searchTerm}) query
  WHERE search_vector @@ query
  ORDER BY rank DESC
  LIMIT 20
`)
```

## 파티셔닝 (대용량 로그 테이블)

```sql
-- Range 파티셔닝 (월별)
CREATE TABLE events (
  id         uuid NOT NULL,
  user_id    uuid NOT NULL,
  event_type text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_01 PARTITION OF events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_2026_02 PARTITION OF events
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

## 유용한 PostgreSQL 함수

```sql
-- UUID 생성 (v7 — 시간순 정렬 가능)
SELECT gen_random_uuid();

-- 통계 쿼리 (N+1 방지)
SELECT
  u.id,
  u.name,
  COUNT(o.id) FILTER (WHERE o.status = 'paid') AS paid_orders,
  COALESCE(SUM(o.total) FILTER (WHERE o.status = 'paid'), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;

-- Upsert
INSERT INTO user_settings (user_id, key, value)
VALUES ($1, $2, $3)
ON CONFLICT (user_id, key)
DO UPDATE SET value = EXCLUDED.value, updated_at = now();

-- CTE (가독성 + 최적화)
WITH active_users AS (
  SELECT id FROM users WHERE deleted_at IS NULL AND is_active = true
),
recent_orders AS (
  SELECT user_id, COUNT(*) AS cnt
  FROM orders
  WHERE created_at > now() - interval '30 days'
  GROUP BY user_id
)
SELECT u.*, COALESCE(r.cnt, 0) AS recent_order_count
FROM active_users u
LEFT JOIN recent_orders r ON r.user_id = u.id;
```

## EXPLAIN ANALYZE 읽기

```sql
-- 실행 계획 확인
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders WHERE user_id = '...' AND status = 'pending';

-- 느린 쿼리 로깅 설정
-- postgresql.conf: log_min_duration_statement = 1000 (1초 이상)
```

## 체크리스트
- [ ] 연결 풀 max는 DB 서버 max_connections의 80% 이내
- [ ] 장기 실행 트랜잭션 모니터링 (pg_stat_activity)
- [ ] VACUUM/ANALYZE 자동 설정 확인
- [ ] 인덱스 사용률 정기 점검 (pg_stat_user_indexes)
