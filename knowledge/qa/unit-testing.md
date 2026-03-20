# 단위 테스트 — 빠르고 독립적인 로직 검증

> 단위 테스트는 코드 문서다. 읽으면 동작 방식이 이해되어야 한다.

## 핵심 원칙
- **FIRST**: Fast, Independent, Repeatable, Self-validating, Timely
- 하나의 테스트는 하나의 동작만 검증한다
- 외부 의존성(DB, API, 파일시스템)은 전부 모킹한다
- 테스트 실행 시간은 개당 50ms 미만을 목표로 한다

## 판단 기준
단위 테스트 작성 대상:
- 순수 함수 (입력 → 출력이 결정론적)
- 비즈니스 규칙과 계산 로직
- 엣지케이스 (null, empty, 경계값, 오버플로우)
- 에러 처리 경로

단위 테스트 제외 대상:
- 단순 getter/setter
- 프레임워크/라이브러리 자체 기능
- 타입 정의만 있는 파일

## 코드 예시 (TypeScript/Vitest)
```typescript
// src/utils/subscription.test.ts
import { describe, it, expect } from 'vitest'
import { calculateNextBillingDate, isTrialExpired } from './subscription'

describe('구독 유틸리티', () => {
  describe('다음 청구일 계산', () => {
    it('월간 구독은 30일 후를 반환한다', () => {
      const startDate = new Date('2026-01-01')
      const result = calculateNextBillingDate(startDate, 'monthly')
      expect(result).toEqual(new Date('2026-01-31'))
    })

    it('연간 구독은 365일 후를 반환한다', () => {
      const startDate = new Date('2026-01-01')
      const result = calculateNextBillingDate(startDate, 'yearly')
      expect(result).toEqual(new Date('2027-01-01'))
    })
  })

  describe('트라이얼 만료 확인', () => {
    it('현재 날짜가 만료일을 지나면 true를 반환한다', () => {
      const expiredDate = new Date('2026-01-01')
      expect(isTrialExpired(expiredDate, new Date('2026-03-20'))).toBe(true)
    })

    it('만료일이 null이면 false를 반환한다', () => {
      expect(isTrialExpired(null, new Date())).toBe(false)
    })
  })
})
```

## 안티패턴
- 여러 동작을 하나의 `it` 블록에서 검증 (단일 책임 위반)
- `expect(true).toBe(true)` 같은 의미 없는 단언
- 테스트 간 공유 상태 (전역 변수, 싱글톤 오염)
- `try/catch`로 에러를 삼키는 테스트

## 실전 팁
- 테스트 이름은 "~일 때 ~해야 한다" 형태로 작성한다
- `beforeEach`에서 상태를 초기화하고 `afterEach`에서 정리한다
- `it.each`로 유사한 케이스를 표 형태로 관리한다
- 실패 메시지가 명확하도록 `expect(...).toBe(...)`에 메시지를 추가한다
