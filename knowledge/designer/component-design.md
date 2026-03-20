# 컴포넌트 설계 — 재사용 가능한 UI의 단위

> 좋은 컴포넌트는 한 가지 일을 잘 한다. 모든 케이스를 혼자 처리하려는 컴포넌트는 괴물이 된다.

## 핵심 원칙

- **단일 책임**: 하나의 컴포넌트는 하나의 역할만
- **조합 가능성**: 컴포넌트끼리 조합하여 더 복잡한 UI를 만들 수 있어야 함
- **변형 (Variants)**: 기능이 같고 모양이 다른 것은 variant로, 기능이 다른 것은 별도 컴포넌트로
- **상태 설계**: Default, Hover, Focus, Active, Disabled, Loading, Error 상태를 반드시 정의
- **콘텐츠 독립성**: 컴포넌트는 내용에 의존하지 않고 구조에 의존해야 함

## 판단 기준

컴포넌트로 만들어야 하는 경우:
- 동일한 UI 패턴이 3회 이상 반복될 때
- 상태(state) 관리가 필요한 인터랙티브 요소
- 재사용 + 일관성 유지가 명확히 요구되는 경우

별도 컴포넌트 vs Variant 결정:
- 동일 목적, 다른 크기/색상 → **Variant**
- 다른 목적, 유사한 외관 → **별도 컴포넌트**

## 실전 예시

**Button 컴포넌트 Variants**
```
Type: Primary / Secondary / Ghost / Danger / Link
Size: sm / md / lg
State: Default / Hover / Focus / Active / Disabled / Loading
Icon: None / Left / Right / Icon-only
```

**컴포넌트 명세 체크리스트**
- [ ] 모든 상태(State) 정의 완료
- [ ] 최소/최대 콘텐츠 길이 처리 (overflow, truncate)
- [ ] 다크모드 대응
- [ ] 접근성 (role, aria, keyboard)
- [ ] 반응형 동작 정의

## 안티패턴

- `Button2`, `ButtonNew`, `ButtonFinal` 등 버전 관리 실패
- 상태 정의 없이 시각 디자인만 완료
- 실제 콘텐츠가 아닌 이상적인 길이의 텍스트만 사용
- 컴포넌트 내부에 여백(margin) 포함 → 외부 레이아웃과 충돌
- 하나의 컴포넌트에 조건부 로직이 너무 많아 Props 폭발

## 실전 팁

- Figma에서 컴포넌트 만들 때 properties panel 반드시 정의
- "이 컴포넌트의 책임은 무엇인가?" 한 문장으로 설명 못 하면 분리
- 컴포넌트 playground 페이지를 별도로 만들어 전체 variant 한눈에 확인
- 개발자와 컴포넌트 API(props) 설계를 함께 협의
- 컴포넌트 변경 전 영향받는 화면 목록 파악 필수

---

## 심화: 컴포넌트 설계 심화 가이드

### Atomic Design 방법론
```
Atoms (원자): Button, Input, Icon, Badge, Avatar
Molecules (분자): Search Bar, Form Field, Card Header
Organisms (유기체): Navbar, Card, Data Table, Form
Templates: Page Layout, Dashboard Shell
Pages: 실제 콘텐츠가 채워진 최종 화면
```

### 컴포넌트 복잡도 관리
**Props 최소화 원칙**
- Boolean props: `isDisabled` → `disabled` (HTML 표준 따르기)
- Variant props: 가능한 한 string enum으로 제한
- Children > 복잡한 content props
- Compound Component 패턴 고려 (Select > Option 관계)

**컴포넌트 분리 신호**
- Props 수가 10개를 넘을 때
- 조건부 렌더링 분기가 3개를 넘을 때
- 두 가지 이상의 역할을 담당할 때
- 특정 부모 컴포넌트에만 의존할 때

### Figma 컴포넌트 구조화
```
[Base] Button
  Properties:
    Type: Primary / Secondary / Ghost / Danger
    Size: sm / md / lg
    State: Default / Hover / Focus / Disabled / Loading
    Icon: None / Left / Right / Only
    
  Variants: 3 types × 3 sizes × 5 states × 4 icon = 최소 핵심 조합
```

### 컴포넌트 문서 템플릿
```markdown
## [컴포넌트명]

### 사용 목적
이 컴포넌트를 사용하는 경우와 사용하지 않아야 하는 경우

### Variants
각 variant의 시각적 예시와 사용 지침

### Props
| Prop | Type | Default | Description |
|------|------|---------|-------------|

### 접근성
키보드 동작, aria 속성, 스크린 리더 지원 방법

### 예시 코드
```

### 컴포넌트 품질 기준
- 모든 상태가 정의되어 있는가?
- 최소/최대 콘텐츠 길이에서 깨지지 않는가?
- 다크모드에서 정상 표시되는가?
- 키보드로 사용 가능한가?
- 스크린 리더로 의미가 전달되는가?
