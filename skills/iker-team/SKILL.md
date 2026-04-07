---
name: iker-team
description: Iker 멀티 에이전트 팀 오케스트레이터. 복합 작업을 분석하고 PO/FE/BE/Designer/QA/Ops/DA/CX/AM 역할을 자동 배분하여 병렬 실행. "팀", "에이전트 팀", "iker" 요청에 사용.
allowed-tools: [Agent, Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, AskUserQuestion, TaskCreate, TaskUpdate, TaskList, TaskGet]
version: 1.0.0
---

# Iker Team — Multi-Agent Orchestrator

Iker 에이전트 팀을 자동으로 편성하고 실행하는 오케스트레이터.

## 역할 카탈로그

| 역할 | 스킬 | 전문 영역 |
|------|------|----------|
| **PO** | iker-po | PRD, 전략, 우선순위, 메트릭 |
| **FE** | iker-fe | 프론트엔드, 컴포넌트, UI |
| **BE** | iker-be | API, DB, 서버, 인프라 |
| **Designer** | iker-designer | UI/UX, 디자인 시스템 |
| **QA** | iker-qa | 테스트, 품질, 코드 리뷰 |
| **Ops** | iker-ops | 프로젝트 관리, 운영 |
| **DA** | iker-da | 데이터 분석, SQL, 시각화 |
| **CX** | iker-cx | VOC 분석, 고객여정, CS, NPS |
| **AM** | iker-am | 세일즈 전략, 피치, 반론 처리, 팔로업, 고객 분석 |

## Workflow

### Phase 1: 작업 분석

사용자의 요청을 분석하여 필요한 역할을 식별한다.

```
"이 기능을 만들어줘" → PO(PRD) + Designer(UI) + FE(구현) + BE(API) + QA(테스트)
"데이터 분석해줘" → DA
"코드 리뷰해줘" → QA + FE/BE
"프로젝트 일정 잡아줘" → Ops + PO
```

### Phase 2: 팀 편성 확인

사용자에게 팀 구성을 보여주고 승인을 받는다.

```
제안 팀 구성:

| # | 역할 | 담당 | 의존성 |
|---|------|------|--------|
| 1 | PO | PRD 작성 | - |
| 2 | Designer | UI 설계 | #1 |
| 3 | FE | 프론트 구현 | #2 |
| 4 | BE | API 구현 | #1 |
| 5 | QA | 테스트 검증 | #3, #4 |

진행할까요?
```

### Phase 3: 에이전트 실행

각 역할의 에이전트를 실행한다. 에이전트 실행 시:

1. 해당 역할의 SKILL.md를 읽어 Identity와 원칙을 주입
2. 태스크에 맞는 Knowledge 파일을 명시적으로 로딩 지시
3. 의존성이 없는 에이전트는 병렬 실행
4. 의존성이 있는 에이전트는 선행 결과를 주입하여 순차 실행

#### 에이전트 프롬프트 구조

```
## Identity
[SKILL.md의 Identity/원칙 섹션 삽입]

## Knowledge
다음 파일을 먼저 읽고 시작하라:
- ~/.claude/knowledge/{role}/{file1}.md
- ~/.claude/knowledge/{role}/{file2}.md

## Context
[프로젝트 배경, 선행 에이전트 결과]

## Goal
[구체적 목표]

## Constraints
[범위 제한, 하지 말아야 할 것]

## Output Format
[기대하는 산출물 형식]
```

### Phase 4: 검증

QA 역할이 각 산출물을 검증한다.
- PASS → Phase 5로
- FAIL → 해당 에이전트에 수정 요청 (최대 3회)

### Phase 5: 결과 보고

```
## 팀 결과

### 산출물
- PO: [PRD 요약]
- Designer: [UI 설계 요약]
- FE: [구현 파일 목록]
- BE: [API 엔드포인트 목록]
- QA: [테스트 결과]

### 검증
- [x] 기능 테스트 통과
- [x] 코드 리뷰 완료
```

## 단일 역할 모드

특정 역할만 필요할 때는 해당 역할의 스킬을 직접 호출:
- `/iker-po PRD 작성해줘`
- `/iker-fe 컴포넌트 리뷰해줘`
- `/iker-da 퍼널 분석해줘`

이 오케스트레이터는 2개 이상의 역할이 협업해야 할 때 사용한다.

## 주의사항

- 단순 작업에 팀을 편성하지 않음 — 단일 역할로 충분하면 해당 스킬 직접 호출
- Knowledge 파일은 태스크에 맞는 것만 선택 로딩 (전체 로딩 금지)
- 각 에이전트의 Boundaries를 존중 — 역할 범위를 넘는 작업 지시 금지
