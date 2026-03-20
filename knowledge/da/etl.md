# ETL / ELT — 데이터 파이프라인 설계와 운영

> 파이프라인은 데이터가 아니라 신뢰를 전달한다. 깨진 파이프라인은 잘못된 의사결정을 만든다.

## 핵심 원칙
- ELT (Extract → Load → Transform): 현대 클라우드 환경의 표준
- 멱등성(Idempotency): 파이프라인은 여러 번 실행해도 동일한 결과여야 한다
- 실패는 예외가 아니라 정상이다 — 재시도, 알림, 롤백 설계
- 데이터 계보(Lineage) 추적: 어디서 왔는지 알아야 수정할 수 있다
- 점진적 로드(Incremental Load)가 전체 재처리보다 효율적

## 판단 기준
| 방식 | 장점 | 단점 | 적합한 경우 |
|------|------|------|-------------|
| Full Refresh | 단순, 항상 최신 | 비용/시간 많음 | 소용량, 변경 추적 불필요 |
| Incremental | 빠름, 저비용 | 복잡, 업데이트 누락 위험 | 대용량, 실시간성 필요 |
| CDC | 변경사항만 캡처 | 인프라 복잡 | 트랜잭션 DB 동기화 |

## 실전 예시

### BigQuery 증분 적재 패턴
```sql
-- MERGE로 Upsert (증분 적재)
MERGE INTO `project.dataset.target_table` AS target
USING (
  SELECT *
  FROM `project.dataset.source_table`
  WHERE updated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
) AS source
ON target.id = source.id
WHEN MATCHED AND source.updated_at > target.updated_at THEN
  UPDATE SET
    target.value = source.value,
    target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
  INSERT (id, value, created_at, updated_at)
  VALUES (source.id, source.value, source.created_at, source.updated_at);
```

### Python ETL 파이프라인 기본 구조
```python
import logging
from datetime import datetime, timedelta
from typing import Optional
import pandas as pd

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class ETLPipeline:
    def __init__(self, name: str):
        self.name = name
        self.run_id = datetime.now().strftime('%Y%m%d_%H%M%S')

    def extract(self, start_date: str, end_date: str) -> pd.DataFrame:
        """소스에서 데이터 추출"""
        logger.info(f"[{self.name}] 추출 시작: {start_date} ~ {end_date}")
        # 실제 구현: API 호출, DB 쿼리 등
        raise NotImplementedError

    def transform(self, df: pd.DataFrame) -> pd.DataFrame:
        """데이터 변환 및 정제"""
        logger.info(f"[{self.name}] 변환 시작: {len(df):,}행")
        raise NotImplementedError

    def load(self, df: pd.DataFrame, table: str, mode: str = 'append'):
        """대상에 데이터 적재"""
        logger.info(f"[{self.name}] 적재 시작: {table} ({mode})")
        raise NotImplementedError

    def run(self, start_date: str, end_date: str):
        try:
            raw = self.extract(start_date, end_date)
            cleaned = self.transform(raw)
            self.load(cleaned, f'{self.name}_table')
            logger.info(f"[{self.name}] 완료: {len(cleaned):,}행 처리")
        except Exception as e:
            logger.error(f"[{self.name}] 실패: {e}")
            self._send_alert(str(e))
            raise

    def _send_alert(self, message: str):
        # Slack, PagerDuty 등 알림
        pass
```

### 파티션 테이블 증분 적재 (BigQuery)
```python
from google.cloud import bigquery

def incremental_load_to_bq(df: pd.DataFrame, project: str,
                            dataset: str, table: str,
                            partition_col: str = 'event_date'):
    client = bigquery.Client(project=project)
    table_ref = f"{project}.{dataset}.{table}"

    # 파티션 지정하여 적재 (해당 파티션만 덮어씀)
    partition_date = df[partition_col].max()
    job_config = bigquery.LoadJobConfig(
        write_disposition='WRITE_TRUNCATE',
        time_partitioning=bigquery.TimePartitioning(
            type_=bigquery.TimePartitioningType.DAY,
            field=partition_col
        )
    )

    destination = f"{table_ref}${partition_date.strftime('%Y%m%d')}"
    job = client.load_table_from_dataframe(df, destination, job_config=job_config)
    job.result()
    logger.info(f"적재 완료: {destination}")
```

## 안티패턴
- 파이프라인에 하드코딩된 날짜 — 항상 파라미터화
- 에러 무시하고 계속 실행 — 잘못된 데이터 누적
- 변환 로직을 적재 코드에 섞기 — 테스트 불가
- 중복 적재 방지 로직 없음 — 재실행 시 데이터 뻥튀기
- 실패 알림 없음 — 파이프라인 죽어도 모름

## 실전 팁
- dbt: 변환 레이어를 SQL로 버전 관리 + 테스트 자동화
- Airflow / Prefect: 의존성 관리, 스케줄링, 재실행
- 데이터 볼륨 모니터링: 갑자기 0건이면 업스트림 장애
- 스테이징 테이블: raw → staging → production 3단계
- 파이프라인 SLA: "9시까지 데이터 준비" 같은 기준 명시
