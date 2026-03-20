# 이벤트 트래킹 — 의미 있는 데이터 수집 설계

> 이벤트 트래킹은 수집이 아니라 질문에서 시작한다. "무엇을 알고 싶은가?"

## 핵심 원칙
- 분석 질문을 먼저 정의하고, 필요한 이벤트를 역으로 설계하라
- 이벤트 네이밍은 일관성이 최우선 (동사_명사 또는 명사_동사)
- 프로퍼티가 없는 이벤트는 반쪽짜리 데이터다
- 클라이언트 기록은 서버 검증보다 신뢰도 낮다
- 이벤트 택소노미(분류 체계)를 문서화하고 팀과 공유하라

## 판단 기준
| 이벤트 유형 | 예시 | 수집 목적 |
|-------------|------|-----------|
| Page View | page_viewed | 콘텐츠 소비 추적 |
| User Action | button_clicked, form_submitted | 행동 패턴 분석 |
| System Event | purchase_completed, error_occurred | 전환/오류 추적 |
| Feature Usage | feature_activated, setting_changed | 기능 채택률 |

## 실전 예시

### 이벤트 네이밍 컨벤션
```
패턴: {객체}_{동사(과거형)}

좋은 예:
- product_viewed          (상품 조회)
- cart_item_added         (장바구니 추가)
- checkout_completed      (결제 완료)
- subscription_started    (구독 시작)
- error_occurred          (오류 발생)

나쁜 예:
- click                   (무엇을 클릭?)
- view_product_page       (동사가 앞에 오면 일관성 깨짐)
- addToCart               (camelCase 혼용)
- event_123               (의미 없음)
```

### 이벤트 스키마 설계
```json
{
  "event_name": "purchase_completed",
  "event_timestamp": "2024-03-15T14:23:45Z",
  "user_id": "usr_abc123",
  "session_id": "ses_xyz789",
  "properties": {
    "order_id": "ord_001",
    "total_amount": 89000,
    "item_count": 3,
    "payment_method": "card",
    "coupon_used": true,
    "coupon_code": "SPRING10"
  },
  "context": {
    "platform": "ios",
    "app_version": "3.2.1",
    "page_path": "/checkout/confirm"
  }
}
```

### Mixpanel/Amplitude 이벤트 전송 (Python)
```python
import mixpanel
from datetime import datetime
import uuid

mp = mixpanel.Mixpanel('YOUR_TOKEN')

def track_purchase(user_id: str, order_data: dict):
    """구매 완료 이벤트 전송"""
    mp.track(user_id, 'purchase_completed', {
        'order_id': order_data['order_id'],
        'total_amount': order_data['amount'],
        'item_count': len(order_data['items']),
        'payment_method': order_data['payment_method'],
        'is_first_purchase': order_data.get('is_first', False),
        # 필수 컨텍스트
        'platform': 'web',
        'timestamp': datetime.now().isoformat()
    })

def identify_user(user_id: str, user_data: dict):
    """사용자 속성 업데이트"""
    mp.people_set(user_id, {
        '$name': user_data['name'],
        '$email': user_data['email'],
        'plan': user_data['plan'],
        'company_size': user_data.get('company_size'),
        'acquisition_channel': user_data.get('channel')
    })
```

### 이벤트 품질 검증 SQL
```sql
-- 이벤트 수 이상 탐지
WITH daily_counts AS (
  SELECT
    event_date,
    event_name,
    COUNT(*) AS event_count,
    COUNT(DISTINCT user_id) AS unique_users,
    COUNTIF(user_id IS NULL) / COUNT(*) AS null_user_rate
  FROM events
  WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  GROUP BY 1, 2
),
event_stats AS (
  SELECT
    event_name,
    AVG(event_count) AS avg_daily_count,
    STDDEV(event_count) AS std_daily_count
  FROM daily_counts
  WHERE event_date < CURRENT_DATE()  -- 오늘 제외
  GROUP BY event_name
)
SELECT
  d.event_date,
  d.event_name,
  d.event_count,
  s.avg_daily_count,
  ABS(d.event_count - s.avg_daily_count) / NULLIF(s.std_daily_count, 0) AS z_score,
  d.null_user_rate
FROM daily_counts d
JOIN event_stats s USING (event_name)
WHERE d.event_date = CURRENT_DATE() - 1
  AND ABS(d.event_count - s.avg_daily_count) / NULLIF(s.std_daily_count, 0) > 3
ORDER BY z_score DESC;
```

## 안티패턴
- 이벤트 이름을 나중에 변경하기 — 기존 데이터와 단절
- 프로퍼티 타입 불일치 (문자열 "1" vs 숫자 1)
- 하나의 이벤트에 너무 많은 정보 — 목적별 분리 이벤트 설계
- 테스트 데이터와 프로덕션 데이터 혼재
- 이벤트 문서화 없이 구현 — 6개월 후 아무도 의미를 모름

## 실전 팁
- 이벤트 택소노미 문서: Notion/Confluence에 이름 + 정의 + 프로퍼티 + 담당자
- 이벤트 검증: 구현 후 실시간 스트림에서 샘플 확인
- 서버사이드 트래킹: 클라이언트 광고 차단에 영향 안 받음
- Super Properties: 모든 이벤트에 공통 속성 자동 첨부 (플랫폼, 버전 등)
- 익명 사용자 ID: 로그인 전/후 연결 (alias/identify 패턴)
