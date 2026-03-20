# 코드 리뷰 — 버그보다 설계를 먼저 검토한다

> 코드 리뷰는 버그 찾기 도구가 아니라 팀 지식 공유와 설계 개선의 장이다.

## 핵심 원칙
- 코드가 아닌 동작을 검토한다
- 지적이 아닌 질문 형태로 피드백한다 ("왜 이렇게 했나요?" 대신 "이 방식의 장점이 무엇인가요?")
- PR 크기는 400줄 이하로 유지한다 (리뷰 효율 급감)
- 자동화할 수 있는 것(린트, 포맷)은 사람이 지적하지 않는다

## 판단 기준
리뷰 우선순위:
1. **보안**: 인증/인가 버그, 입력 검증 미흡
2. **정확성**: 비즈니스 로직 오류, 엣지케이스 누락
3. **성능**: N+1 쿼리, 불필요한 루프, 메모리 누수
4. **설계**: 결합도, 단일 책임, 확장성
5. **가독성**: 네이밍, 주석, 복잡도

승인 기준:
- [ ] 요구사항을 충족하는가?
- [ ] 테스트가 충분한가?
- [ ] 엣지케이스를 처리했는가?
- [ ] 문서/주석이 필요한 부분에 있는가?

## 코드 예시 (리뷰 코멘트 패턴)
```typescript
// 리뷰어가 발견한 문제 — N+1 쿼리
// Before (문제 있는 코드)
async function getOrdersWithUsers(orderIds: string[]) {
  const orders = await Order.findAll({ where: { id: orderIds } })
  // N+1 발생: orders.length만큼 쿼리 실행
  return Promise.all(orders.map(order => order.getUser()))
}

// After (리뷰 후 개선)
async function getOrdersWithUsers(orderIds: string[]) {
  return Order.findAll({
    where: { id: orderIds },
    include: [{ model: User }],  // JOIN으로 단일 쿼리
  })
}

// 테스트 커버리지 누락 지적 예시
// 리뷰어: "해당 함수의 amount가 0인 경우 처리가 어떻게 되나요?
// 테스트가 없는 것 같아서요."
it('결제 금액이 0원인 경우 에러를 반환한다', () => {
  expect(() => processPayment({ amount: 0 })).toThrow('유효하지 않은 금액')
})
```

## 안티패턴
- "이 코드 잘못됨" 같은 감정적 지적
- 1000줄 이상 PR을 24시간 이내 리뷰 요청
- 스타일 문제를 수동으로 지적 (Prettier/ESLint가 할 일)
- LGTM(Looks Good To Me) 없이 승인

## 실전 팁
- PR 설명에 "어떤 문제를 왜 이렇게 해결했는가"를 먼저 작성한다
- Conventional Comments(`suggestion:`, `nitpick:`, `question:`)로 중요도를 구분한다
- 리뷰는 24시간 이내에 첫 응답을 보장한다 (팀 SLA)
- 반복되는 패턴은 린트 룰이나 코드 템플릿으로 자동화한다
