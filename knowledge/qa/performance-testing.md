# 성능 테스트 — 느린 앱은 동작하지 않는 앱과 같다

> 성능 버그는 늦게 발견될수록 수정 비용이 기하급수적으로 늘어난다.

## 핵심 원칙
- 성능 기준(SLA)을 먼저 정의하고 테스트한다
- 부하 테스트, 스트레스 테스트, 스파이크 테스트를 구분한다
- 병목은 추정하지 말고 측정한다
- CI에 성능 회귀 테스트를 포함시킨다

## 판단 기준
| 테스트 유형 | 목적 | 도구 |
|------------|------|------|
| 부하 테스트 | 예상 트래픽 처리 확인 | k6, Artillery |
| 스트레스 테스트 | 한계점 파악 | k6 |
| 스파이크 테스트 | 급격한 트래픽 증가 대응 | k6 |
| 소크 테스트 | 장시간 안정성 확인 | k6 |

SLA 기준 예시:
- API P95 응답시간 < 300ms
- 페이지 로드 < 3초 (LCP)
- 동시 접속 100명 시 에러율 < 0.1%

## 코드 예시 (k6 + TypeScript)
```typescript
// performance/api-load-test.js (k6 스크립트)
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate } from 'k6/metrics'

const errorRate = new Rate('errors')

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // 워밍업
    { duration: '5m', target: 100 },  // 목표 부하
    { duration: '2m', target: 0 },    // 쿨다운
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],  // P95 300ms 미만
    errors: ['rate<0.01'],             // 에러율 1% 미만
  },
}

export default function () {
  const token = `Bearer ${__ENV.TEST_TOKEN}`

  const response = http.get('http://api.example.com/subscriptions', {
    headers: { Authorization: token },
  })

  const success = check(response, {
    '200 응답': (r) => r.status === 200,
    '응답 시간 300ms 미만': (r) => r.timings.duration < 300,
    '올바른 스키마': (r) => r.json('data') !== undefined,
  })

  errorRate.add(!success)
  sleep(1)
}
```

## 안티패턴
- 성능 테스트를 릴리즈 직전에만 실행
- 프로덕션 환경에서 스트레스 테스트 실행
- 데이터베이스 쿼리 성능 무시하고 앱 서버만 테스트
- N+1 쿼리를 프론트엔드 최적화로 해결하려는 시도

## 실전 팁
- `EXPLAIN ANALYZE`로 느린 쿼리를 먼저 찾는다
- Lighthouse CI를 GitHub Actions에 연동해 프론트엔드 성능 회귀를 자동 탐지한다
- 성능 테스트 전용 데이터셋(프로덕션 규모)을 미리 준비한다
- 결과를 Grafana 대시보드에 연동해 트렌드를 추적한다
