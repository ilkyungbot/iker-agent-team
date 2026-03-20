# 다크 모드 — 어두운 환경을 위한 체계적 설계

> 다크 모드는 색상 반전이 아니다. 완전히 새로운 컬러 시스템을 설계하는 것이다.

## 핵심 원칙

- **색상 반전 금지**: 단순 invert는 대비비 역전과 채도 문제를 일으킨다
- **표면 계층**: 어두운 배경에서 elevation은 밝기로 표현한다
- **채도 조정**: 다크 배경에서 채도 높은 색상은 진동 효과(vibration)를 일으킨다
- **독립적 토큰**: 라이트/다크 각각의 시맨틱 토큰으로 관리
- **시스템 설정 연동**: prefers-color-scheme 미디어 쿼리 지원

## 판단 기준

**다크 모드 배경 계층 (Material Design 참고)**
```
Surface levels (어두울수록 낮은 층, 밝을수록 높은 층):
  surface-base:    #121212 (최하단, 가장 어두움)
  surface-1:       #1E1E1E (기본 카드)
  surface-2:       #232323 (raised 카드)
  surface-3:       #252525 (모달, 드로어)
  surface-4:       #272727 (툴팁 등 최상위)
```

**다크 모드 대비비 기준**
- 텍스트: 7:1 이상 권장 (라이트 기준 4.5:1보다 높게)
- 이유: 어두운 배경에서 눈이 더 민감하게 반응

## 실전 예시

**컬러 토큰 듀얼 정의**
```
color-text-primary:
  light: #111827
  dark:  #F9FAFB

color-surface-default:
  light: #FFFFFF
  dark:  #1E1E1E

color-brand-primary:
  light: #3B82F6 (Blue 500)
  dark:  #60A5FA (Blue 400, 채도 낮추고 명도 높임)
```

**다크 모드 이미지/일러스트**
- 사진: 밝기 살짝 감소 (brightness 90%), 오버레이 없이
- 로고/아이콘: 다크 버전 별도 제작 또는 white 버전 사용
- 스크린샷: 다크 UI 스크린샷으로 대체

## 안티패턴

- `filter: invert(1)` 전체 적용
- 라이트 모드와 동일한 Primary 색상 다크에 그대로 사용
- 다크 배경에 순수 흰색(#FFFFFF) 텍스트 → 눈 피로 (off-white 권장)
- 그림자를 다크 모드에서도 동일하게 적용 (어둠 위에 어둠)
- 다크 모드 테스트를 최후에 한 번만 진행

## 실전 팁

- Figma Variables로 light/dark 컬러 모드 전환 구현 (Mode 기능)
- 다크 모드는 신규 기능 개발 시 함께 설계. 나중에 추가하면 비용 2배
- OLED 화면 대응: 완전 검정(#000000) 배경 옵션 제공 가능
- 이미지가 많은 화면은 다크 모드에서 별도 검토 필수
- 사용자 설정 > 시스템 설정 순으로 우선순위 부여

---

## 심화: 다크 모드 구현 가이드

### Figma Variables로 다크 모드 구현
```
1. Local Variables 패널 열기
2. Color 컬렉션 생성
3. Mode 추가: "Light" / "Dark"
4. 각 변수에 Light/Dark 값 각각 입력
5. 컴포넌트에서 hex 대신 Variable 참조
6. 프레임 선택 후 Mode 전환으로 즉시 확인
```

### 코드 구현 방법
**CSS Custom Properties 방식**
```css
:root {
  --color-bg: #FFFFFF;
  --color-text: #111827;
}

[data-theme='dark'] {
  --color-bg: #1E1E1E;
  --color-text: #F9FAFB;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme='light']) {
    --color-bg: #1E1E1E;
    --color-text: #F9FAFB;
  }
}
```

### 다크 모드 이미지 처리
**CSS filter 활용**
```css
@media (prefers-color-scheme: dark) {
  img:not([src*=".svg"]) {
    filter: brightness(0.85);
  }
}
```

**picture 엘리먼트로 다크 버전 제공**
```html
<picture>
  <source srcset="logo-dark.svg" media="(prefers-color-scheme: dark)">
  <img src="logo-light.svg" alt="로고">
</picture>
```

### 다크 모드 QA 체크리스트
- [ ] 모든 텍스트의 대비비 WCAG AA 통과
- [ ] 이미지/아이콘 다크 배경에서 적절히 표시
- [ ] 그림자가 다크에서 elevation으로 대체됨
- [ ] 시스템 설정 변경 시 즉시 전환 (새로고침 불필요)
- [ ] 사용자 설정 저장 (localStorage)
- [ ] OS 다크모드 자동 감지 동작 확인
- [ ] 인쇄 시 라이트 모드로 강제 전환

### 다크 모드 특수 케이스
**차트/데이터 시각화**
- 라이트: 밝은 배경 + 어두운 색 계열
- 다크: 어두운 배경 + 채도 낮추고 밝기 올린 색 계열
- 그리드 라인: 다크에서는 더 투명하게 (opacity 0.1~0.2)

**코드 블록**
- 다크 모드에서는 높은 대비의 신택스 하이라이팅
- 인기 있는 다크 테마: Dracula, One Dark, Tokyo Night
