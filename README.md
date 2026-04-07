# Iker Agent Team

AI 에이전트 팀 스킬셋 for Claude Code. 9개 시니어 역할 + 오케스트레이터 + 170개 Knowledge 파일.

## 역할

| 스킬 | 역할 | Knowledge |
|------|------|-----------|
| `iker-po` | Product Owner | 24개 |
| `iker-fe` | Frontend Engineer | 25개 |
| `iker-be` | Backend Engineer | 25개 |
| `iker-designer` | Product Designer | 24개 |
| `iker-qa` | QA Engineer | 24개 |
| `iker-ops` | Operations Lead | 24개 |
| `iker-da` | Data Analyst | 24개 |
| `iker-cx` | Customer Experience | - |
| `iker-am` | Account Manager | - |
| `iker-team` | 오케스트레이터 | - |

## 설치

```bash
# 스킬 복사
cp -r skills/iker-* ~/.claude/skills/

# Knowledge 복사
cp -r knowledge/* ~/.claude/knowledge/
```

## 사용법

```
# 단일 역할
/iker-po PRD 작성해줘
/iker-da 퍼널 분석해줘

# 팀 편성
/iker-team 이 기능 만들어줘
```

## 구조

각 에이전트는 SOUL(정체성/원칙) + Knowledge(판단 기준/코드 예시/안티패턴)로 구성됩니다. Knowledge는 태스크별로 온디맨드 로딩되어 토큰을 절약합니다.

## 영감

[IronAct AI 에이전트 팀 실전 구축 가이드](https://ironact.gitbook.io/ironact-docs/CrEhRPJQJpia3xh9iqbi)의 멀티 에이전트 시스템을 Claude Code 스킬로 재구성했습니다.

## 라이선스

MIT

