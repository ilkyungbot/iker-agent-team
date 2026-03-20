# 아이콘 디자인 — 의미를 담은 최소한의 시각 언어

> 좋은 아이콘은 설명 없이도 의미가 전달된다. 설명이 필요하다면 아이콘 대신 텍스트를 쓰라.

## 핵심 원칙

- **명확성**: 아이콘이 무엇을 의미하는지 즉각 인식되어야 한다
- **일관성**: 동일한 스타일, 두께, 크기 그리드를 따른다
- **단순성**: 세부 사항을 줄이고 본질적 형태만 남긴다
- **맥락 의존성**: 아이콘은 항상 레이블 또는 컨텍스트와 함께 사용
- **확장성**: 16px에서도 24px에서도 명확하게 보여야 한다

## 판단 기준

**아이콘 크기 기준**
| 크기 | 사용처 |
|------|--------|
| 16px | 인라인 텍스트, 배지 내부 |
| 20px | 컴팩트 UI, 리스트 아이템 |
| 24px | 기본 UI 컴포넌트, 버튼 |
| 32px | 강조 요소, 빈 상태 보조 |
| 48px+ | 일러스트레이션, 온보딩 |

**아이콘 스타일 선택**
- Outline: 가볍고 모던. 일반 상태
- Filled: 무겁고 강조. 선택/활성 상태
- 혼용: Outline(기본) + Filled(선택) → 상태 변화 시각화

## 실전 예시

**아이콘 그리드 설계 (24px 기준)**
```
총 크기: 24×24px
라이브 영역: 20×20px (padding 2px)
안전 영역: 16×16px (복잡한 형태는 이 안에)
획 두께: 1.5px (outline 스타일 기준)
모서리: 2px radius (일관된 부드러움)
```

**아이콘 명명 규칙**
- `icon-[category]-[name]`
- `icon-nav-home`, `icon-action-add`, `icon-status-error`
- Filled 버전: `icon-action-add-filled`

## 안티패턴

- 여러 스타일의 아이콘 혼용 (획 두께, 모서리 반경 불일치)
- 아이콘만으로 의미 전달 시도 (특히 추상적 개념)
- 텍스트를 포함한 아이콘 (현지화 불가)
- 픽셀 경계에 걸친 아이콘 (흐릿하게 렌더링)
- 아이콘에 불필요한 그라데이션, 그림자

## 실전 팁

- 커스텀 아이콘 전 Heroicons, Lucide, Phosphor 등 오픈소스 활용
- Figma에서 아이콘은 컴포넌트로 등록. Boolean 속성으로 Filled/Outline 전환
- SVG export 시 `currentColor` 사용으로 컬러 변경 유연성 확보
- 아이콘 세트는 한 사람이 디자인하거나, 정확한 스타일 가이드 기반으로 일관성 유지
- 신규 아이콘 추가 전 기존 아이콘으로 표현 가능한지 먼저 검토

---

## 심화: 아이콘 시스템 구축 가이드

### 아이콘 인식성 테스트
**5초 테스트**
- 아이콘만 보여주고 의미를 물어봄
- 70% 이상이 맞추면 통과
- 실패 시: 레이블 추가 또는 아이콘 교체

**친숙한 아이콘 vs 커스텀 아이콘**
- 친숙한: 돋보기(검색), 집(홈), 하트(좋아요) → 레이블 없어도 인식
- 커스텀: 기업 고유 기능 아이콘 → 반드시 레이블 병용

### 아이콘 제작 가이드 (Figma)
**그리드 설정 (24px 기준)**
```
Frame: 24×24px
Grid: 1px 단위
Center guides: 12, 12
Padding: 2px (전체 안전 영역)
Live area: 20×20px (핵심 형태 영역)
```

**획 일관성 체크**
- Stroke width: 1.5px (24px 아이콘 기준)
- Stroke Cap: Round
- Stroke Join: Round
- 완성 후 Outline Stroke (확장 필수)

**모서리 처리**
- 외각 모서리: 2px radius
- 내부 모서리: 1px radius (눈에 띄지 않게)
- 직각이 필요한 경우만 0 radius 사용

### 오픈소스 아이콘 라이브러리 비교
| 라이브러리 | 아이콘 수 | 스타일 | 라이선스 |
|-----------|---------|--------|---------|
| Heroicons | 292 | Outline/Solid | MIT |
| Lucide | 1400+ | Outline | ISC |
| Phosphor | 9000+ | 6가지 스타일 | MIT |
| Material Icons | 2500+ | 5가지 스타일 | Apache 2.0 |
| Feather | 287 | Outline | MIT |

### 아이콘 접근성 구현
```html
<!-- 의미 있는 아이콘 버튼 -->
<button aria-label="파일 삭제">
  <svg aria-hidden="true" focusable="false">...</svg>
</button>

<!-- 텍스트와 함께 사용되는 아이콘 -->
<span>
  <svg aria-hidden="true">...</svg>
  저장하기
</span>

<!-- 상태를 나타내는 아이콘 -->
<svg role="img" aria-label="완료">...</svg>
```

### 아이콘 애니메이션
- 로딩: 회전 스피너 (360deg, 1s, linear, infinite)
- 상태 전환: Filled ↔ Outline (scale + opacity)
- 방향 전환: 화살표 회전 (180deg, 200ms)
- 알림: 흔들기 (shake, 300ms, ease-in-out)
