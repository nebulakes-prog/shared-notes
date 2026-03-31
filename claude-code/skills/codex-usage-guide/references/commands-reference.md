# Codex Commands Reference

## /codex:review

표준 읽기 전용 코드 리뷰.

### 옵션
| 옵션 | 설명 |
|------|------|
| `--base <ref>` | 비교 기준 브랜치/커밋 (예: `--base main`) |
| `--scope auto\|working-tree\|branch` | 리뷰 범위 |
| `--wait` | 포그라운드 실행 (결과까지 대기) |
| `--background` | 백그라운드 실행 |

### 사용 예시
```bash
/codex:review                        # 미커밋 변경사항 리뷰
/codex:review --base main            # main 브랜치 대비 전체 변경 리뷰
/codex:review --background           # 백그라운드에서 리뷰
/codex:review --scope working-tree   # 워킹 트리만 리뷰
```

### 주의사항
- 리뷰 전용. 코드 수정 불가.
- 커스텀 리뷰 지시가 필요하면 `/codex:adversarial-review` 사용.
- 자동으로 리뷰 크기 추정 후 포그라운드/백그라운드 권장.

---

## /codex:adversarial-review

설계 결정에 도전하는 비판적 리뷰. 가정, 트레이드오프, 실패 시나리오 검토.

### 옵션
| 옵션 | 설명 |
|------|------|
| `--base <ref>` | 비교 기준 |
| `--scope auto\|working-tree\|branch` | 리뷰 범위 |
| `--wait` | 포그라운드 실행 |
| `--background` | 백그라운드 실행 |
| `[focus text]` | 집중 검토할 영역 (자연어) |

### 사용 예시
```bash
/codex:adversarial-review                              # 전체 도전적 리뷰
/codex:adversarial-review --base main                  # 브랜치 전체
/codex:adversarial-review challenge caching design     # 캐싱 설계 집중 검토
/codex:adversarial-review --background look for race conditions
```

### /codex:review와의 차이
| 구분 | review | adversarial-review |
|------|--------|--------------------|
| 초점 | 구현 결함, 버그 | 설계 결정, 가정, 트레이드오프 |
| 톤 | 중립적 | 도전적 (의문 제기) |
| 커스텀 포커스 | X | O (자연어 추가 가능) |
| 검토 영역 | 코드 품질 | 인증, 데이터 손실, 경합 조건, 대안적 접근 |

---

## /codex:rescue

Codex에 작업을 위임. 디버깅, 구현, 조사, 수정 모두 가능.

### 옵션
| 옵션 | 설명 |
|------|------|
| `--background` | 백그라운드 실행 |
| `--wait` | 포그라운드 실행 |
| `--resume` | 이전 Codex 작업 이어서 |
| `--fresh` | 새로운 작업으로 시작 (이전 컨텍스트 무시) |
| `--model <name>` | 모델 지정 (기본: 설정 따름) |
| `--effort <level>` | 추론 수준: none, minimal, low, medium, high, xhigh |
| `--write` | 쓰기 모드 (기본값) |

### 모델 옵션
| 모델 | 용도 |
|------|------|
| (기본) | 설정의 기본 모델 사용 |
| `gpt-5.4-mini` | 빠르고 저렴한 작업 |
| `spark` (= `gpt-5.3-codex-spark`) | 가장 빠르고 가벼운 패스 |
| 기타 모델명 | 직접 지정 가능 |

### 사용 예시
```bash
# 디버깅
/codex:rescue investigate why tests are failing
/codex:rescue fix the test with smallest patch

# 이어서 작업
/codex:rescue --resume apply top fix from last run
/codex:rescue --resume dig deeper into the auth issue

# 모델/성능 제어
/codex:rescue --model spark fix quickly
/codex:rescue --model gpt-5.4-mini --effort medium investigate flaky test

# 백그라운드
/codex:rescue --background investigate regression in payment module

# 자연어 위임 (슬래시 커맨드 없이)
"Ask Codex to redesign the database connection to be more resilient."
```

### 주의사항
- 기본적으로 쓰기 모드 (`--write`). 읽기만 원하면 명시적으로 요청.
- 작업이 길면 `--background` 권장.
- 실패 시 Claude가 대신 답변하지 않음.

---

## /codex:status

실행 중이거나 최근 완료된 작업 상태 확인.

```bash
/codex:status                    # 모든 작업 상태
/codex:status task-abc123        # 특정 작업 상태
```

---

## /codex:result

완료된 작업의 최종 결과물 표시. Codex 세션 ID 포함.

```bash
/codex:result                    # 최근 완료 작업 결과
/codex:result task-abc123        # 특정 작업 결과
```

세션 ID로 Codex에서 직접 재개 가능:
```bash
codex resume <session-id>
```

---

## /codex:cancel

활성 백그라운드 작업 취소.

```bash
/codex:cancel                    # 최근 작업 취소
/codex:cancel task-abc123        # 특정 작업 취소
```

---

## /codex:setup

Codex 설치 및 인증 상태 확인.

```bash
/codex:setup                         # 상태 확인
/codex:setup --enable-review-gate    # 리뷰 게이트 활성화
/codex:setup --disable-review-gate   # 리뷰 게이트 비활성화
```

### Review Gate
- 활성화 시: Claude 응답마다 자동으로 Codex 리뷰 실행.
- **경고**: Claude-Codex 루프로 사용량 급증 가능. 사람 감독 필수.

---

## 서브에이전트: codex:codex-rescue

Agent tool에서 `subagent_type: "codex:codex-rescue"`로 사용 가능.
Claude Code가 막혔을 때, 두 번째 구현/진단 패스가 필요할 때, 심층 근본 원인 조사가 필요할 때 proactively 사용.

```typescript
Agent({
  subagent_type: "codex:codex-rescue",
  prompt: "investigate why the auth middleware is rejecting valid tokens",
  run_in_background: true
})
```

### 사용 기준
- Claude 메인 스레드가 상당한 디버깅/구현 작업을 넘겨야 할 때 **사전적으로** 사용.
- Claude가 빠르게 끝낼 수 있는 단순 작업은 넘기지 않음.
