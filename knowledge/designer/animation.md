# 애니메이션 — 움직임으로 의미를 전달하는 기술

> 좋은 애니메이션은 보이지 않는다. 사용자가 자연스럽다고 느끼면 성공이다.

## 핵심 원칙

- **목적 중심**: 애니메이션은 미적 장식이 아닌 의미 전달·피드백·방향 제시용
- **물리 기반**: 실제 물리 법칙(관성, 가속/감속)을 따를수록 자연스럽다
- **속도**: UI 애니메이션은 일반적으로 150~400ms. 너무 느리면 답답함
- **Easing**: Linear는 로봇처럼 보인다. ease-in-out 또는 spring 계열 사용
- **접근성 존중**: `prefers-reduced-motion` 설정 사용자에게는 애니메이션 제거

## 판단 기준

| 애니메이션 유형 | 권장 시간 | Easing |
|--------------|---------|--------|
| 버튼 hover/active | 80~150ms | ease-out |
| 모달 열기/닫기 | 200~300ms | ease-in-out |
| 페이지 전환 | 250~350ms | ease-in-out |
| Toast/Notification | 200~300ms enter, 150ms exit | ease-out |
| Skeleton → Content | 300~400ms | ease-in |
| 드래그 이동 | 0ms (즉각) | - |

## 실전 예시

**방향성 있는 트랜지션**
- 계층이 깊어질 때: 오른쪽으로 슬라이드 in
- 뒤로 갈 때: 왼쪽으로 슬라이드 out
- 모달/팝오버: 스케일업(scale) + fade in
- 목록 아이템 추가: 아래서 위로 fade-slide in

**Spring 애니메이션 (자연스러움의 핵심)**
```
// Framer Motion 예시
transition: { type: "spring", stiffness: 300, damping: 30 }
// CSS 예시
transition: transform 250ms cubic-bezier(0.34, 1.56, 0.64, 1);
```

## 안티패턴

- 모든 요소에 애니메이션 적용 → 주의가 분산되고 성능 저하
- Linear easing 사용 → 기계적이고 어색함
- 삭제 후 즉시 사라짐 (undo 기회 없음)
- 애니메이션 완료 전 다음 액션 불가 → 사용자를 기다리게 함
- 움직임 과다증 (motion sickness 유발 가능)

## 실전 팁

- Figma Smart Animate + Auto Layout으로 빠른 프로토타이핑
- After Effects → Lottie로 복잡한 일러스트 애니메이션 export
- 애니메이션 검토는 반드시 실제 기기에서 (Figma mirror 활용)
- 팀 내 "애니메이션 없이도 UX가 완성되는가?" 기준 먼저 충족
- CSS will-change 속성으로 GPU 가속 활용, transform/opacity만 애니메이션

---

## 심화: 애니메이션 시스템 구축

### 모션 토큰 정의
```
duration:
  instant:  0ms    (상태 변화, 색상 전환)
  fast:     100ms  (hover, 소규모 UI 변화)
  normal:   200ms  (컴포넌트 등장/퇴장)
  slow:     300ms  (모달, 페이지 전환)
  xslow:    500ms  (온보딩, 강조 애니메이션)

easing:
  standard:    cubic-bezier(0.4, 0.0, 0.2, 1)  (기본)
  decelerate:  cubic-bezier(0.0, 0.0, 0.2, 1)  (등장)
  accelerate:  cubic-bezier(0.4, 0.0, 1, 1)     (퇴장)
  sharp:       cubic-bezier(0.4, 0.0, 0.6, 1)   (빠른 전환)
```

### 애니메이션 결정 프레임워크
**이 애니메이션이 필요한가?**
1. 사용자에게 공간 방향성을 알려주는가?
2. 상태 변화의 원인-결과를 명확히 하는가?
3. 시스템이 작동 중임을 알려주는가?
4. 브랜드 개성을 적절히 전달하는가?

하나라도 해당하면 사용, 아니면 제거.

### 접근성: prefers-reduced-motion
```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```
- 사용자 시스템 설정에서 활성화 가능
- 전정 장애, 편두통 환자에게 중요
- 애니메이션 완전 제거 or 대체 (fade only)

### Lottie 활용 가이드
- After Effects → Bodymovin 플러그인 → JSON export
- 파일 크기 최적화: 불필요한 레이어 제거, 벡터 단순화
- Loop vs OneShot: 로딩 = loop, 완료 = one-shot
- 색상 변경이 필요하면 Lottie Properties로 런타임 제어

### 성능 최적화
- 애니메이션 대상: transform, opacity만 사용 (GPU 가속)
- Layout thrashing 유발 속성 피하기: width, height, top, left
- will-change: transform 신중하게 사용 (메모리 비용)
- requestAnimationFrame으로 JS 애니메이션 최적화
