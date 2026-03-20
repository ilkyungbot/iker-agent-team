# Domain-Driven Design

## 핵심 개념

| 개념 | 설명 |
|------|------|
| Entity | 고유 식별자가 있는 객체 (User, Order) |
| Value Object | 불변, 속성으로 동일성 판단 (Money, Address) |
| Aggregate | 일관성 경계를 가진 Entity 클러스터 |
| Aggregate Root | Aggregate의 단일 진입점 |
| Domain Event | 도메인에서 발생한 중요한 사건 |
| Repository | Aggregate 영속성 추상화 |

## Entity 구현

```typescript
abstract class Entity<T extends { id: string }> {
  protected readonly props: T

  constructor(props: T) {
    this.props = props
  }

  get id() { return this.props.id }

  equals(other: Entity<T>): boolean {
    return this.id === other.id
  }
}

class User extends Entity<UserProps> {
  static create(props: CreateUserProps): User {
    return new User({
      id:        generateId(),
      email:     Email.create(props.email),
      name:      props.name,
      role:      UserRole.MEMBER,
      createdAt: new Date()
    })
  }

  activate() {
    if (this.props.isActive) return
    this.props.isActive = true
    this.addDomainEvent(new UserActivatedEvent(this.id))
  }
}
```

## Value Object 구현

```typescript
class Money {
  private constructor(
    private readonly amount:   number,
    private readonly currency: string
  ) {
    if (amount < 0) throw new Error('Amount cannot be negative')
  }

  static of(amount: number, currency: string): Money {
    return new Money(Math.round(amount), currency)
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch')
    return Money.of(this.amount + other.amount, this.currency)
  }

  multiply(factor: number): Money {
    return Money.of(Math.round(this.amount * factor), this.currency)
  }

  get value() { return this.amount }
  get currencyCode() { return this.currency }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency
  }
}
```

## Domain Event

```typescript
abstract class DomainEvent {
  readonly occurredAt: Date
  readonly eventId:    string

  constructor() {
    this.occurredAt = new Date()
    this.eventId    = generateId()
  }
}

class OrderPlacedEvent extends DomainEvent {
  constructor(
    readonly orderId: string,
    readonly userId:  string,
    readonly total:   Money
  ) { super() }
}

// Aggregate에서 이벤트 발행
class Order extends Entity<OrderProps> {
  private domainEvents: DomainEvent[] = []

  place(): void {
    this.props.status = 'placed'
    this.domainEvents.push(
      new OrderPlacedEvent(this.id, this.props.userId, this.props.total)
    )
  }

  pullDomainEvents(): DomainEvent[] {
    const events = [...this.domainEvents]
    this.domainEvents = []
    return events
  }
}
```

## 바운디드 컨텍스트

```
[User Context]     [Order Context]    [Payment Context]
  User                Order               Payment
  Profile             OrderItem           Invoice
  UserRepo            OrderRepo           PaymentRepo
       |                   |                   |
       └────── Domain Events ──────────────────┘
              (UserCreated, OrderPlaced, PaymentCompleted)
```

## 체크리스트
- [ ] 바운디드 컨텍스트 경계 명확히 정의
- [ ] Aggregate 내부 직접 접근 금지 (Root 통해서만)
- [ ] Domain Event로 컨텍스트 간 통신
- [ ] Value Object는 불변으로 구현
- [ ] 도메인 로직은 서비스 레이어가 아닌 엔티티에
