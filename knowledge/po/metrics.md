# 메트릭 — AARRR, North Star, Leading/Lagging
> 핵심 메시지: "측정하지 않으면 개선할 수 없고, 잘못된 것을 측정하면 잘못된 방향으로 최적화된다"

## 핵심 원칙
- North Star Metric: 제품의 핵심 가치를 대표하는 단 하나의 지표
- Leading Indicator: 미래 성과를 예측하는 지표 (선행 지표)
- Lagging Indicator: 과거 성과를 확인하는 지표 (후행 지표)
- 가드레일 지표: 핵심 지표 개선 중 악화되면 안 되는 지표

## 판단 기준
- 이 지표가 실제 고객 가치를 반영하는가?
- 팀이 이 지표에 직접 영향을 줄 수 있는가?
- 지표 조작이 어렵고 의미 있는 행동만으로 개선되는가?
- 주간 또는 월간으로 추적하기에 적합한가?

## 실전 예시 (프레임워크)

### North Star Metric 정의법
1. 핵심 고객 가치 액션 식별 (Aha Moment)
2. 그 액션의 빈도 또는 완료율을 지표화
3. 비즈니스 성장과의 상관관계 검증
4. 전사 공유 및 정기 리뷰

예시 North Star Metrics:
- Spotify: Monthly Active Users listening 30+ min/day
- Airbnb: Nights booked
- Slack: Teams sending 2,000+ messages

### AARRR 지표 (Pirate Metrics)
| 단계 | 의미 | 핵심 지표 예시 |
|-----|------|-------------|
| Acquisition | 사용자 획득 | 신규 방문자, CAC |
| Activation | 첫 가치 경험 | 온보딩 완료율, Aha Moment |
| Retention | 재방문/재사용 | DAU/MAU, 리텐션 곡선 |
| Revenue | 수익화 | MRR, ARPU, LTV |
| Referral | 바이럴 | NPS, 추천 전환율 |

### 지표 계층 구조
```
North Star Metric (제품 핵심 가치)
├── L1 지표 (사업 성과: MRR, DAU)
│   ├── L2 지표 (제품 행동: 기능 사용률)
│   │   └── L3 지표 (세부 이벤트: 클릭률)
```

### OKR과 지표 연결
- Objective: 정성적 목표 방향
- Key Result: 측정 가능한 결과 (지표 + 목표치)
- Initiative: KR 달성을 위한 구체적 활동

예시:
- O: 사용자 활성화 경험을 획기적으로 개선한다
- KR1: 온보딩 완료율 45% → 65%
- KR2: D7 리텐션 25% → 35%

### 리텐션 분석 방법
- 코호트 분석: 같은 기간 가입자의 시간에 따른 리텐션 변화
- L28 리텐션: 28일 중 활성 일수
- 자연 리텐션: 광고 없이도 돌아오는 사용자 비율

## 안티패턴
- Vanity Metrics: 페이지뷰, 다운로드 수 → 실제 가치와 무관
- 너무 많은 지표 추적: "Everything is a priority = nothing is"
- 단기 지표만 최적화: 장기 사용자 가치 무시
- 지표 목표치 조작: KR 달성을 위한 기준 낮추기
- 데이터 없는 직관만으로 의사결정

## 실전 팁
- 주간 지표 리뷰를 팀 루틴으로 정착 (15-30분)
- 지표 이상 감지 시 즉시 원인 분석 프로세스 마련
- A/B 테스트 결과를 지표 변화와 연결
- 지표 대시보드를 팀 전체가 볼 수 있게 공개
- 분기말에 지표 목표 달성 여부와 학습 내용 기록
