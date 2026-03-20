# 디자인 토큰 — 디자인과 코드를 연결하는 공통 언어

> 토큰은 변수가 아니다. 디자인 결정을 코드로 표현한 의미의 단위다.

## 핵심 원칙

- **시맨틱 이름**: 색상값이 아닌 역할 이름으로 (blue-500 X, color-primary O)
- **3단계 계층**: Primitive → Semantic → Component
- **플랫폼 독립성**: 동일한 토큰이 Web, iOS, Android에서 동일한 의미
- **단일 출처**: Figma Variables 또는 Style Dictionary 기반 단일 관리
- **변경 추적**: 토큰 변경은 Changelog에 기록

## 판단 기준

**3단계 토큰 계층**
```
1. Primitive (원시값)
   blue-50: #EFF6FF
   blue-500: #3B82F6
   space-4: 4px
   
2. Semantic (의미)
   color-primary: {blue-500}
   color-text-default: {gray-900}
   space-component-gap: {space-4}
   
3. Component (컴포넌트)
   button-primary-background: {color-primary}
   button-padding-x: {space-4}
```

## 실전 예시

**토큰 카테고리 분류**
```
color/
  text/, background/, border/, icon/
  status/ (success, warning, error, info)
  brand/ (primary, secondary)

spacing/
  component/ (내부 padding, gap)
  layout/ (섹션 간격, page margin)

typography/
  font-family/, font-size/, font-weight/, line-height/

elevation/
  shadow-sm, shadow-md, shadow-lg

motion/
  duration-fast: 150ms
  duration-normal: 250ms
  easing-standard: cubic-bezier(0.4, 0, 0.2, 1)

radius/
  sm: 4px, md: 8px, lg: 16px, full: 9999px
```

## 안티패턴

- 토큰 없이 컴포넌트에 하드코딩된 hex 값 사용
- Primitive 토큰을 컴포넌트에 직접 참조 (blue-500을 버튼에 직접)
- 토큰 이름에 스타일 값 포함 (`padding-16px` → 값이 바뀌면 이름도 바뀜)
- Figma와 코드 토큰 싱크 없이 각자 관리
- 토큰 변경 후 영향 범위 확인 없이 배포

## 실전 팁

- Style Dictionary, Theo, Tokens Studio 플러그인으로 자동화
- Figma Variables (2023~)로 디자이너가 직접 토큰 관리 가능
- 토큰 변경 PR은 디자이너와 개발자 공동 리뷰
- JSON 형식으로 토큰 export → 모든 플랫폼에서 공유
- 첫 도입 시: 컬러 토큰부터 시작. 전체를 한번에 하려다 중단되는 경우 多

---

## 심화: 디자인 토큰 심화 가이드

### Style Dictionary 구성 예시
```json
// tokens/color/primitive.json
{
  "color": {
    "blue": {
      "50": { "value": "#EFF6FF" },
      "500": { "value": "#3B82F6" },
      "900": { "value": "#1E3A8A" }
    }
  }
}

// tokens/color/semantic.json
{
  "color": {
    "primary": { "value": "{color.blue.500}" },
    "text": {
      "default": { "value": "{color.gray.900}" },
      "muted": { "value": "{color.gray.500}" }
    }
  }
}
```

### 플랫폼별 토큰 변환
Style Dictionary 변환 예시:
```
입력: JSON 토큰 정의
출력:
  Web: CSS Custom Properties (--color-primary: #3B82F6)
  iOS: Swift (ColorTokens.primary = UIColor(hex: "3B82F6"))
  Android: XML (colorPrimary: #3B82F6)
  Figma: Variables (자동 동기화)
```

### Tokens Studio (Figma 플러그인) 워크플로우
1. Tokens Studio 플러그인 설치
2. JSON 토큰 파일 연결 (GitHub 동기화 가능)
3. Figma 스타일/변수에 자동 동기화
4. 개발자는 같은 JSON으로 코드 토큰 생성
5. 토큰 변경 → PR → 디자이너/개발자 공동 리뷰

### 토큰 버저닝 전략
```
Semantic Versioning 적용:
  Major (1.0.0 → 2.0.0): 하위 호환 불가 변경 (토큰명 변경)
  Minor (1.0.0 → 1.1.0): 신규 토큰 추가
  Patch (1.0.0 → 1.0.1): 값만 변경 (버그 수정)
```

### 토큰 마이그레이션 전략
레거시 코드에서 점진적 토큰 도입:
1. 새로운 컴포넌트부터 토큰 사용
2. 수정이 발생하는 기존 코드에 토큰 적용
3. 전체 감사(Audit) 후 하드코딩 일괄 교체
4. Linting 규칙으로 하드코딩 방지

### 토큰 네이밍 Best Practice
```
패턴: [category]-[concept]-[variant]-[state]

예시:
  color-bg-primary-hover
  color-text-inverse-default
  spacing-component-gap-md
  typography-body-size-lg
  shadow-elevation-high
  radius-component-button
  motion-duration-fast
```
