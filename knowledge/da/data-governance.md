# 데이터 거버넌스 — 믿을 수 있는 데이터 환경 만들기

> 거버넌스는 규제가 아니라 신뢰 인프라다. 데이터를 믿을 수 있어야 의사결정을 믿을 수 있다.

## 핵심 원칙
- 데이터 오너십을 명확히 하라 — 아무도 책임지지 않는 데이터는 썩는다
- 데이터 카탈로그: "이 테이블이 무엇인가?"에 30초 안에 답할 수 있어야 한다
- 접근 권한은 최소 권한 원칙으로 — 필요한 사람만 필요한 데이터에
- 개인정보는 처음부터 설계에 반영하라 (Privacy by Design)
- 지표 정의 불일치가 가장 흔한 신뢰 파괴 요인이다

## 판단 기준
| 거버넌스 영역 | 핵심 질문 | 도구 |
|---------------|-----------|------|
| 데이터 카탈로그 | 이 데이터가 무엇인가? | dbt docs, DataHub, Dataplex |
| 데이터 계보 | 어디서 왔는가? | dbt lineage, OpenLineage |
| 품질 관리 | 신뢰할 수 있는가? | Great Expectations, dbt test |
| 접근 제어 | 누가 볼 수 있는가? | BigQuery IAM, Column-level security |
| 지표 관리 | 정의가 일치하는가? | Metric Store (dbt Metrics, Looker) |

## 실전 예시

### dbt 메타데이터 문서화
```yaml
# models/marts/fct_orders.yml
version: 2

models:
  - name: fct_orders
    description: |
      주문 팩트 테이블. 결제 완료된 모든 주문 포함.
      취소/환불 건은 order_status로 구분 (cancelled, refunded).
      파티션: order_date (일별)
    meta:
      owner: data-team@company.com
      team: analytics
      sla: D+1 09:00 KST
    columns:
      - name: order_id
        description: 주문 고유 ID (ord_ 접두사)
        tests:
          - not_null
          - unique
      - name: user_id
        description: 구매자 user_id (dim_users 참조)
        tests:
          - not_null
          - relationships:
              to: ref('dim_users')
              field: user_id
      - name: net_revenue
        description: |
          순매출 = 총매출 - 할인 - 환불.
          VAT 포함 금액. 단위: 원(KRW).
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### 지표 단일화 (Metric Store)
```yaml
# metrics/revenue.yml (dbt Metrics)
metrics:
  - name: monthly_revenue
    label: 월간 순매출
    description: |
      월별 결제 완료 주문의 순매출 합계.
      취소/환불 건 제외. VAT 포함.
    model: ref('fct_orders')
    calculation_method: sum
    expression: net_revenue
    timestamp: order_date
    time_grains: [month, quarter, year]
    filters:
      - field: order_status
        operator: '='
        value: "'completed'"
    dimensions:
      - channel
      - country
      - product_category
```

### BigQuery 컬럼 수준 접근 제어
```sql
-- 개인정보 컬럼 마스킹
CREATE OR REPLACE VIEW `project.masked_dataset.users_masked` AS
SELECT
  user_id,
  -- PII 마스킹
  CONCAT(LEFT(email, 2), '***@***.com') AS email_masked,
  LEFT(phone, 3) AS phone_prefix,
  -- 비PII
  country,
  user_segment,
  created_at
FROM `project.raw.users`;

-- 역할별 접근 정책
-- 분석팀: masked view만 접근
-- 데이터팀: 원본 접근
-- 외부 파트너: 집계 테이블만 접근
```

### 데이터 거버넌스 체크리스트
```
테이블 등록 시:
□ 테이블 설명 (용도, 갱신 주기, 담당자)
□ 컬럼 설명 (정의, 단위, 허용값)
□ 데이터 품질 테스트 (not_null, unique, referential integrity)
□ SLA 정의 (언제까지 준비되어야 하는가)
□ PII 포함 여부 및 마스킹 정책
□ 접근 권한 설정 (누가 볼 수 있는가)
□ Upstream/Downstream 의존성 문서화
```

## 안티패턴
- 지표 정의 없이 대시보드 배포 — 부서마다 다른 숫자
- 퇴사자 테이블 오너십 방치 — 아무도 모르는 좀비 테이블
- 개인정보 필드를 분석 편의상 모든 팀에 열어두기
- 문서화를 "나중에" — 나중은 오지 않는다
- 거버넌스를 데이터팀만의 일로 여기기 — 전사 책임

## 실전 팁
- 데이터 계약(Data Contract): 업스트림 팀이 스키마 변경 시 사전 통보
- Slack 알림 연동: 품질 테스트 실패 시 즉시 담당자에게 알림
- 분기별 데이터 감사: 미사용 테이블, 오래된 문서 정리
- 거버넌스 KPI: 문서화율, 품질 테스트 커버리지, PII 위반 건수
- 데이터 메시(Data Mesh): 도메인별 자율 거버넌스 + 연방 표준
