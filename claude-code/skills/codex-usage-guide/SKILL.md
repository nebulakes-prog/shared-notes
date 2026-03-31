---
name: codex-usage-guide
description: Use when deciding whether to delegate work to Codex, when unsure how to use Codex commands, or when a task matches Codex strengths (code review, debugging, second opinion, background delegation). Triggers on Codex, code review delegation, stuck debugging, second implementation pass, rescue.
---

# Codex Usage Guide

OpenAI Codex plugin for Claude Code. Codex = GPT-5.4 기반 코딩 에이전트.
로컬 Codex CLI를 통해 동작하며, Claude Code 안에서 작업 위임/리뷰/디버깅에 사용.

## Version Check (하루 1회 자동 실행)

스킬 활성화 시 `_meta.json`의 `last_checked`와 오늘 날짜를 비교한다.
**같은 날이면 체크를 건너뛰고** 바로 본문으로 진행한다.
다른 날이면 아래 2단계를 실행한다.

### Step 1: GitHub 최신 버전 vs 설치된 플러그인 버전

원격 최신 버전 확인:
```bash
gh api repos/openai/codex-plugin-cc/releases/latest --jq '.tag_name' 2>/dev/null
```
releases가 없으면:
```bash
gh api repos/openai/codex-plugin-cc/contents/plugins/codex/.claude-plugin/plugin.json --jq '.content' | base64 -d | jq -r '.version'
```

설치된 버전 확인:
```bash
cat ~/.claude/plugins/marketplaces/openai-codex/plugins/codex/.claude-plugin/plugin.json
```

비교:
- **같으면** → 플러그인 최신 상태. Step 2로.
- **다르면** → 새 버전 존재.
  1. 사용자에게 알림: "Codex 플러그인 새 버전(X.Y.Z)이 있습니다."
  2. `_meta.json`의 `auto_update`가 `true`이면 → 묻지 않고 바로 업데이트 진행
  3. `auto_update`가 `false`이면 → 사용자에게 업데이트 여부 확인
  4. 플러그인 업데이트: `/plugin update codex@openai-codex` (또는 사용자에게 실행 안내)
  5. 업데이트 후 바로 Step 2의 "다름" 경로로 진행 (사용가이드도 업데이트)

### Step 2: 설치된 플러그인 버전 vs 사용가이드 버전

```
~/.claude/plugins/marketplaces/openai-codex/plugins/codex/.claude-plugin/plugin.json의 version
  vs
_meta.json의 codex_plugin_version
```

- **같으면** → 가이드가 최신 상태. `_meta.json`의 `last_checked`를 오늘로 갱신. 끝.
- **다르면** → 플러그인이 업데이트됐는데 가이드가 안 맞음.
  1. 플러그인의 `CHANGELOG.md`와 새로 추가/변경된 commands, skills, agents를 확인
  2. 공식 GitHub README에서 변경사항 확인 (WebFetch)
  3. 가이드 파일들(SKILL.md, references/*.md) 업데이트
  4. `_meta.json`의 `codex_plugin_version`과 `last_checked`를 갱신

### auto_update 설정

`_meta.json`의 `auto_update` 필드:
- `true` → 플러그인 업데이트 + 가이드 업데이트를 사용자 확인 없이 자동 진행
- `false` → 매 단계에서 사용자에게 확인 후 진행
- 사용자가 "자동 업데이트 켜줘/꺼줘"라고 하면 이 값을 변경

### 체크 실패 시

gh CLI 미설치, 네트워크 불가, 파일 경로 변경 등으로 체크가 실패하면:
- 에러를 무시하고 현재 가이드로 진행. 작업을 블로킹하지 않는다.
- "버전 체크를 건너뛰었습니다"로 간단히 알림. `last_checked`는 갱신하지 않는다 (다음 활성화 시 재시도).

## When to Use Codex (의사결정 매트릭스)

### USE Codex

| 상황 | 명령어 | 이유 |
|------|--------|------|
| 코드 리뷰 (커밋 전/PR 전) | `/codex:review` | 독립적 시각의 리뷰 |
| 설계 결정 도전/검증 | `/codex:adversarial-review` | 가정과 트레이드오프 테스트 |
| 디버깅에 막힘 | `/codex:rescue` | 다른 AI 관점으로 돌파 |
| 단일 파일 집중 수정 | `/codex:rescue` | 한 가지를 깊이 파는 데 강점 |
| 서브에이전트 루틴 작업 | `codex:codex-rescue` agent | Sonnet 대체, Claude 토큰 절약 |
| 멀티모델 브레인스토밍 | Opus + Codex 병렬 | 다양한 관점 확보 |
| 장시간 백그라운드 작업 | `/codex:rescue --background` | Claude는 다른 작업 계속 |

### DO NOT use Codex

| 상황 | 이유 |
|------|------|
| 복잡한 아키텍처/전략 판단 | Opus가 더 적합 |
| 멀티파일 대규모 리팩토링 | Claude 메인 컨텍스트가 더 효과적 |
| 보안 심층 분석 | security-reviewer agent 사용 |
| Claude가 빠르게 끝낼 수 있는 단순 작업 | 위임 오버헤드 불필요 |
| 사용자가 즉시 대화형 피드백 원할 때 | Codex는 비동기 |

## Command Quick Reference

| 명령어 | 용도 | 모드 |
|--------|------|------|
| `/codex:review` | 표준 코드 리뷰 | 읽기 전용 |
| `/codex:adversarial-review` | 도전적 리뷰 (설계 검증) | 읽기 전용 |
| `/codex:rescue` | 작업 위임 (디버깅, 구현, 조사) | 읽기+쓰기 |
| `/codex:status` | 실행 중/최근 작업 상태 | 조회 |
| `/codex:result` | 완료된 작업 결과 | 조회 |
| `/codex:cancel` | 백그라운드 작업 취소 | 제어 |
| `/codex:setup` | 설치/인증 상태 확인 | 설정 |

상세 명령어 옵션: [commands-reference.md](references/commands-reference.md)

## Decision Flowchart

```
작업 발생
  |
  v
Claude가 직접 빠르게 (< 2분) 처리 가능?
  |-- YES --> Claude가 직접 처리
  |-- NO
      |
      v
  코드 리뷰/검증 작업?
      |-- YES --> 표준 리뷰? --> /codex:review
      |           설계 도전? --> /codex:adversarial-review
      |-- NO
          |
          v
      디버깅/구현/조사 작업?
          |-- YES --> /codex:rescue
          |           (작업 크면 --background 추가)
          |-- NO
              |
              v
          아키텍처/전략/보안?
              |-- YES --> Opus agent 사용
              |-- NO --> Claude 메인에서 처리
```

## Key Rules

1. **리뷰 결과 후 자동 수정 금지**: `/codex:review` 결과를 받은 후 반드시 사용자에게 어떤 이슈를 수정할지 물어볼 것. 자동으로 코드 수정하지 않음.
2. **루프 주의**: Review Gate 활성화 시 Claude-Codex 루프로 사용량 급증 가능. 사람 감독 하에서만 사용.
3. **백그라운드 권장**: 작업이 클 경우 `--background`로 실행하고 `/codex:status`로 진행률 확인.
4. **출력 보존**: Codex 출력은 그대로 전달. 요약/의역하지 않음.
5. **실패 시 대체 금지**: Codex 실행 실패 시 Claude가 대신 답변하지 않음. 실패를 보고하고 중단.
6. **프롬프트 품질**: Codex는 대화 맥락을 모른다. 요청에 파일 경로, 에러, 제약 조건, 실패한 시도를 명시적으로 포함할 것. [prompting-guide.md](references/prompting-guide.md) 참조.

## Typical Workflows

### 1. 배포 전 리뷰
```
/codex:review                    # 미커밋 변경사항 리뷰
/codex:review --base main        # 브랜치 전체 리뷰
```

### 2. 막힌 디버깅 위임
```
/codex:rescue investigate why tests are failing
/codex:rescue fix the test with smallest patch
```

### 3. 장기 백그라운드 작업
```
/codex:rescue --background investigate regression in auth module
/codex:status                    # 진행률 확인
/codex:result                    # 완료 후 결과 확인
```

### 4. 이전 작업 이어서
```
/codex:rescue --resume apply top fix from last run
```

### 5. 가벼운 모델로 빠르게
```
/codex:rescue --model spark fix quickly
/codex:rescue --model gpt-5.4-mini --effort medium investigate flaky test
```

## Setup

```bash
# 인증 확인
/codex:setup

# 로그인 필요 시
!codex login

# 리뷰 게이트 (선택, 주의해서 사용)
/codex:setup --enable-review-gate
/codex:setup --disable-review-gate
```

## Configuration

Codex 설정 파일:
- 사용자: `~/.codex/config.toml`
- 프로젝트: `.codex/config.toml`

```toml
model = "gpt-5.4-mini"              # 기본 모델
model_reasoning_effort = "xhigh"     # 추론 수준
```
