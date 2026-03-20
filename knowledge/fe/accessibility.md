# Accessibility — 웹 접근성

> 접근성은 특정 사용자를 위한 것이 아니다. 모든 상황의 모든 사용자를 위한 것이다.

## 핵심 원칙
- **시맨틱 HTML 우선**: div/span 대신 button, nav, main, article
- **키보드 완전 조작**: 마우스 없이도 모든 기능 사용 가능
- **ARIA는 마지막**: 네이티브 HTML로 먼저 해결, 안 될 때만 ARIA
- **색상만으로 정보 전달 금지**: 색맹 사용자를 위해 아이콘/텍스트 병행

## WCAG 2.1 AA 기준
| 기준 | 요구사항 |
|------|----------|
| 색상 대비 | 일반 텍스트 4.5:1, 대형 텍스트 3:1 |
| 포커스 표시 | 키보드 포커스 시 시각적 표시 필수 |
| 텍스트 크기 | 200% 확대해도 기능 유지 |
| 터치 타겟 | 최소 44x44px |

## 코드 예시

❌ 의미 없는 div 버튼
```tsx
<div onClick={handleSubmit} className="btn">제출</div>
// 키보드 포커스 없음, 스크린리더가 버튼으로 인식 불가
```

✅ 시맨틱 버튼
```tsx
<button type="submit" onClick={handleSubmit}>제출</button>
```

❌ 아이콘만 있는 버튼
```tsx
<button><SearchIcon /></button>
```

✅ 접근 가능한 아이콘 버튼
```tsx
<button aria-label="검색">
  <SearchIcon aria-hidden="true" />
</button>
```

✅ 모달 접근성
```tsx
function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const closeRef = useRef<HTMLButtonElement>(null)

  useEffect(() => {
    if (isOpen) closeRef.current?.focus()
  }, [isOpen])

  return (
    <dialog
      open={isOpen}
      aria-labelledby="modal-title"
      aria-modal="true"
      onKeyDown={(e) => e.key === 'Escape' && onClose()}
    >
      <h2 id="modal-title">{title}</h2>
      {children}
      <button ref={closeRef} onClick={onClose}>닫기</button>
    </dialog>
  )
}
```

✅ 폼 레이블 연결
```tsx
<div>
  <label htmlFor="email">이메일 *</label>
  <input
    id="email"
    type="email"
    aria-required="true"
    aria-describedby="email-error"
  />
  {error && (
    <span id="email-error" role="alert">{error}</span>
  )}
</div>
```

## 실전 팁
- axe DevTools 브라우저 확장으로 자동 검사
- `Tab` 키로 직접 네비게이션 테스트
- VoiceOver(Mac) 또는 NVDA(Windows)로 스크린리더 테스트
- shadcn/ui의 Radix UI 기반 컴포넌트는 접근성 기본 내장

## 주의사항
- `tabindex="0"` 남용 — 자연스러운 DOM 순서가 우선
- `aria-label` 중복 — label이 있으면 aria-label 불필요
- `outline: none` 금지 — 포커스 링을 제거하지 말 것 (커스텀 스타일은 OK)
