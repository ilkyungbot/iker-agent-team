# 모바일 UX — 손 위의 경험을 위한 설계

> 모바일은 PC의 축소판이 아니다. 완전히 다른 맥락과 행동 패턴을 가진 환경이다.

## 핵심 원칙

- **엄지 존 (Thumb Zone)**: 화면 하단 중앙이 가장 편안한 영역. 핵심 액션을 배치
- **단순화**: 한 화면에 하나의 핵심 목적. 멀티태스킹 UI 지양
- **터치 피드백**: 모든 터치 가능 요소에 시각적·촉각적 반응
- **스크롤 중심**: 모달보다 스크롤, 탭 전환보다 인라인 콘텐츠 선호
- **맥락 인식**: 위치, 카메라, 알림 등 모바일 고유 기능을 적극 활용

## 판단 기준

**엄지 존 분류**
```
[쉬운 도달 영역] - 화면 하단 70%
[보통 영역]     - 화면 중앙
[어려운 영역]   - 화면 상단 (중요 정보 표시용, 액션 금지)
```

**터치 타겟 크기**
- 최소: 44×44px (Apple HIG), 48×48dp (Material Design)
- 권장: 56×56px 이상
- 타겟 간 최소 간격: 8px

## 실전 예시

**모바일 네비게이션 패턴**
- Tab Bar (4~5개): 주요 섹션 간 이동, 항상 노출
- Bottom Sheet: 컨텍스트 액션 메뉴 (iOS Action Sheet 대체)
- Floating Action Button: 주요 생성 액션 1개
- Swipe Navigation: 탭/카드 간 수평 스와이프

**입력 최적화**
- 숫자 입력: `inputmode="numeric"` → 숫자 키패드 자동 표시
- 이메일 입력: `type="email"` → @ 키 노출
- 자동 완성: autocomplete 어트리뷰트 명세 제공
- 긴 폼: 단계별 분할 (Step 1/3 형태)

## 안티패턴

- Hover에만 나타나는 정보 (터치 기기에서 hover 없음)
- 작은 텍스트 링크로만 네비게이션 (탭 실패율 높음)
- 풀스크린 모달 과다 사용 → 뒤로가기 맥락 붕괴
- 화면 상단에 주요 CTA 배치
- 데스크톱에서만 테스트하고 출시

## 실전 팁

- iOS와 Android 각각의 Design Guideline (HIG, Material 3) 숙지 필수
- Safe Area (notch, home indicator) 항상 고려
- 단방향 스크롤 원칙: 한 화면에서 세로+가로 스크롤 동시 사용 지양
- 앱 vs 웹뷰 차이 이해: 앱은 제스처 활용 가능, 웹뷰는 제한적
- 테스트: iPhone SE (소형), iPhone Pro Max (대형), 구형 Android 3종 필수

---

## 심화: 모바일 UX 심화 가이드

### iOS vs Android 주요 차이
| 항목 | iOS | Android |
|------|-----|---------|
| 뒤로가기 | 스와이프 or 상단 Back 버튼 | 시스템 뒤로가기 버튼 |
| 탭바 위치 | 하단 고정 | 하단 권장 (Material 3) |
| 시트 | Bottom Sheet | Bottom Sheet |
| 알림 스타일 | iOS 시스템 스타일 | Material 알림 스타일 |
| 폰트 | SF Pro | Roboto (기본) |

### Safe Area 처리
```
iPhone의 Safe Area:
  - Status Bar: 44~59px (노치/다이나믹 아일랜드)
  - Home Indicator: 34px (홈 인디케이터)
  
Android의 Safe Area:
  - Status Bar: 24dp
  - Navigation Bar: 48dp (제스처 모드 시 약 21dp)
  
CSS:
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
```

### 모바일 제스처 설계
- **탭**: 단일 요소 선택/활성화
- **롱 프레스**: 컨텍스트 메뉴, 멀티 선택 진입
- **스와이프 좌우**: 삭제, 아카이브 (리스트 아이템)
- **스와이프 아래**: 새로고침 (Pull to Refresh)
- **핀치**: 줌 인/아웃 (이미지, 지도)
- **제스처 중복 주의**: 시스템 제스처와 충돌 방지

### 모바일 성능 고려사항
- 이미지: WebP 포맷, 레이지 로딩
- 폰트: 서브셋, 시스템 폰트 우선 고려
- 애니메이션: 60fps 유지, 저사양 기기 배려
- 오프라인 상태 처리: 연결 없을 때 동작 정의

### 모바일 테스트 매트릭스
```
필수 테스트 기기:
  iOS: iPhone SE (소형), iPhone 15 Pro Max (대형)
  Android: Galaxy S23 (고사양), 저가형 Android

테스트 항목:
  - 세로/가로 모드 전환
  - 키보드 올라올 때 레이아웃 변화
  - 다이나믹 타입 (큰 글자 설정)
  - 저전력 모드
  - 느린 3G 네트워크
```
