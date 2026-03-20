# Database Design

## Drizzle ORM 스키마 정의

```typescript
import { pgTable, uuid, text, timestamp, boolean, integer, index, uniqueIndex } from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

export const users = pgTable('users', {
  id:        uuid('id').primaryKey().defaultRandom(),
  email:     text('email').notNull(),
  name:      text('name').notNull(),
  role:      text('role', { enum: ['admin', 'member'] }).notNull().default('member'),
  isActive:  boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp('deleted_at', { withTimezone: true })
}, (table) => ({
  emailIdx:     uniqueIndex('users_email_idx').on(table.email),
  createdAtIdx: index('users_created_at_idx').on(table.createdAt)
}))

export const orders = pgTable('orders', {
  id:        uuid('id').primaryKey().defaultRandom(),
  userId:    uuid('user_id').notNull().references(() => users.id, { onDelete: 'restrict' }),
  status:    text('status', { enum: ['pending', 'paid', 'cancelled'] }).notNull().default('pending'),
  total:     integer('total').notNull(),  // 원 단위 정수 저장 (소수점 오류 방지)
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow()
}, (table) => ({
  userIdIdx: index('orders_user_id_idx').on(table.userId),
  statusIdx: index('orders_status_idx').on(table.status)
}))

// Relations 정의
export const usersRelations = relations(users, ({ many }) => ({
  orders: many(orders)
}))

export const ordersRelations = relations(orders, ({ one }) => ({
  user: one(users, { fields: [orders.userId], references: [users.id] })
}))
```

## Drizzle 쿼리 패턴

```typescript
import { eq, and, or, desc, isNull, inArray } from 'drizzle-orm'

// 기본 CRUD
const user = await db.select().from(users).where(eq(users.id, userId)).limit(1)
await db.insert(users).values({ email, name }).returning()
await db.update(users).set({ name, updatedAt: new Date() }).where(eq(users.id, id))

// Soft delete
await db.update(users).set({ deletedAt: new Date() }).where(eq(users.id, id))

// 활성 사용자만 조회 (soft delete 적용)
const activeUsers = await db.select()
  .from(users)
  .where(isNull(users.deletedAt))

// JOIN
const usersWithOrders = await db.select({
  user: users,
  orderCount: count(orders.id)
})
  .from(users)
  .leftJoin(orders, eq(users.id, orders.userId))
  .groupBy(users.id)

// 트랜잭션
await db.transaction(async (tx) => {
  const [order] = await tx.insert(orders).values({ userId, total }).returning()
  await tx.update(users).set({ updatedAt: new Date() }).where(eq(users.id, userId))
  return order
})
```

## 네이밍 컨벤션

| 항목 | 규칙 | 예시 |
|------|------|------|
| 테이블 | snake_case 복수 | `user_profiles` |
| 컬럼 | snake_case | `created_at`, `is_active` |
| PK | `id` (uuid) | `id uuid DEFAULT gen_random_uuid()` |
| FK | `{table_singular}_id` | `user_id`, `order_id` |
| 인덱스 | `{table}_{columns}_idx` | `users_email_idx` |
| Unique 인덱스 | `{table}_{columns}_uidx` | `users_email_uidx` |

## 금액/수량 저장 규칙
- **금액**: 최소 단위 정수 (원화 → 원 단위 integer)
- **비율**: `numeric(5,4)` (소수점 4자리)
- **날짜**: 항상 `timestamptz` (timezone 포함)

## 체크리스트
- [ ] 모든 테이블에 `created_at`, `updated_at` 포함
- [ ] Soft delete가 필요한 엔티티는 `deleted_at` 추가
- [ ] FK에 적절한 onDelete 정책 설정 (restrict/cascade)
- [ ] 자주 조회되는 컬럼에 인덱스 추가
- [ ] 금액은 정수 저장 (float 금지)
