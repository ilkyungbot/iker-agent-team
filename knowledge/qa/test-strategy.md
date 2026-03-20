# 테스트 전략 — 무엇을, 얼마나, 어떤 순서로 테스트할 것인가

> 테스트는 버그를 잡는 도구가 아니라 소프트웨어 설계의 피드백 루프다.

## 핵심 원칙
- **Testing Trophy**: 정적 분석 > 단위 테스트 > 통합 테스트 > E2E (Kent C. Dodds 모델)
- 테스트는 사용자 행동을 중심으로 작성한다 (구현 세부사항 테스트 금지)
- 빠른 피드백 루프: 로컬 < 1분, CI < 10분
- 테스트 코드도 프로덕션 코드와 동일한 품질 기준을 적용한다

## 판단 기준
| 레이어 | 비중 | 목적 |
|--------|------|------|
| 단위 | 40% | 비즈니스 로직 검증 |
| 통합 | 40% | 모듈 간 협력 검증 |
| E2E | 15% | 크리티컬 사용자 여정 |
| 정적 분석 | 5% | 타입/린트 |

- 리스크 기반 우선순위: 결제 > 인증 > 핵심 기능 > 부가 기능
- 변경 빈도가 높은 코드일수록 테스트 밀도를 높인다

## 코드 예시 (TypeScript/Vitest)
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    coverage: {
      provider: 'v8',
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80,
      },
      exclude: ['**/types/**', '**/*.d.ts', '**/index.ts'],
    },
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
  },
})
```

## 안티패턴
- 커버리지 숫자만 맞추기 위한 의미 없는 테스트 작성
- 구현 세부사항(내부 함수명, 상태 변수)에 의존하는 테스트
- 단위 테스트로 통합 문제를 해결하려 하기
- 테스트 없이 "나중에 추가하겠다" 선언

## 실전 팁
- 테스트 작성이 어렵다면 설계 문제다 — 리팩터링 신호로 읽어라
- `describe` 블록은 "주어진 상황에서"로, `it`은 "~해야 한다"로 시작한다
- 테스트 파일은 소스 파일과 동일한 디렉터리에 위치시킨다
- 실패하는 테스트를 먼저 커밋하고 구현한다 (TDD)
