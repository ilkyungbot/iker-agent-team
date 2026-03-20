# 타이포그래피 — 텍스트가 곧 인터페이스다

> 좋은 타이포그래피는 읽히지 않는다. 사용자가 내용에만 집중하게 만든다.

## 핵심 원칙

- **계층 (Hierarchy)**: 크기·굵기·색상으로 정보 중요도를 시각화한다
- **가독성 (Readability)**: 긴 텍스트는 행간 1.5~1.7, 줄 길이 60~80자가 최적
- **일관성**: 폰트 스타일은 제한적으로 (보통 2종류, 최대 3종류)
- **스케일 시스템**: 임의 크기보다 4pt/8pt 기반 타입 스케일 사용
- **대비**: 배경 대비 최소 4.5:1 (일반 텍스트), 3:1 (대형 텍스트)

## 판단 기준

| 역할 | 크기 가이드 | 굵기 |
|------|------------|------|
| Display | 40px~72px | 700~900 |
| Heading 1 | 28px~36px | 600~700 |
| Heading 2 | 20px~24px | 600 |
| Body | 14px~16px | 400 |
| Caption | 12px | 400~500 |
| Label | 12px~14px | 500~600 |

## 실전 예시

**타입 스케일 정의 (8pt 기반)**
```
xs: 12px / line-height: 16px
sm: 14px / line-height: 20px
base: 16px / line-height: 24px
lg: 18px / line-height: 28px
xl: 20px / line-height: 28px
2xl: 24px / line-height: 32px
3xl: 30px / line-height: 36px
```

**폰트 페어링**
- 제목 (Serif or Display) + 본문 (Sans-serif): 클래식하고 신뢰감
- 단일 폰트 패밀리의 굵기 변화: 심플하고 모던

## 안티패턴

- 한 화면에 5가지 이상 폰트 크기 혼용
- 줄 간격을 1.0으로 설정해 가독성 저하
- 전체 대문자(ALL CAPS) 남용 → 스크린 리더 문제 + 가독성 저하
- 배경과 유사한 색상의 텍스트 (저대비)
- 모바일에서 12px 미만 폰트 사용

## 실전 팁

- 타입 스케일은 Figma Variable 또는 Text Style로 관리. 임의 값 금지
- 프로토타입 단계부터 실제 콘텐츠 길이로 테스트 (Lorem ipsum 함정)
- 한국어는 영문 대비 행간을 10~20% 더 넓게 설정
- Weight 변화로 계층을 표현할 때, 두 단계 이상 차이를 줘야 인식됨
- 영문·숫자 혼용 시 한글 폰트와 영문 폰트를 별도 지정 (font-family stack)

---

## 심화: 타이포그래피 심화 가이드

### 한국어 타이포그래피 특수사항
- **행간**: 한국어는 영문 대비 1.2~1.4배 넓게 설정 (권장 1.7~2.0)
- **자간**: 한국어는 0 또는 약간 넓게 (-1em~0)
- **줄 맞춤**: 왼쪽 정렬 기본. 양쪽 맞춤은 단어 간격 왜곡 주의
- **폰트 선택**: Pretendard, Noto Sans KR, Spoqa Han Sans Neo 등
- **혼합 사용**: `font-family: 'Pretendard', 'Inter', sans-serif` (한글 → 영문 → 시스템)

### Variable Font 활용
- 하나의 폰트 파일로 weight 1~999 모두 표현
- Pretendard Variable, Inter Variable 등 지원
- 성능: 여러 weight 파일 대신 단일 파일로 절감
- CSS: `font-variation-settings: 'wght' 450`

### 읽기 최적화 원칙
```
본문 텍스트 최적 조건:
  - 폰트 크기: 16~18px
  - 행간: 1.5~1.7
  - 줄 길이: 60~80자 (약 600~800px)
  - 대비: #333 on white (또는 토큰 사용)
  - 자간: 0 (body), 약간 넓게 (caption, label)
```

### 타입 스케일 선택 방법
**Modular Scale (비율 기반)**
- 비율 1.25 (Major Third): 소규모 UI, 차이가 미묘함
- 비율 1.333 (Perfect Fourth): 균형 잡힌 일반 앱
- 비율 1.5 (Perfect Fifth): 강한 계층, 마케팅/랜딩

**도구**: type-scale.com, Utopia (유동적 타입 스케일)

### 특수 상황 타이포그래피
**숫자 표현**
- 가격/통계: 타뷸러 숫자(font-variant-numeric: tabular-nums)로 열 정렬
- 큰 숫자: 천 단위 구분 쉼표, 단위 명시 (원, %, 명)

**긴 텍스트 처리**
- Truncation: 한 줄 `-webkit-line-clamp: 1`
- 여러 줄: `-webkit-line-clamp: 3`
- Tooltip으로 전체 텍스트 표시

**강조 텍스트**
- Bold(700): 중요 키워드 강조
- Italic: 한국어에는 잘 사용하지 않음
- Underline: 링크에만 사용. 강조 목적 사용 자제
