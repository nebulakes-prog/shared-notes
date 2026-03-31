# shared-notes

Things I found useful while using Claude Code and AI tools.

## codex-usage-guide

Claude Code에서 OpenAI Codex를 효과적으로 사용하기 위한 스킬입니다.

Codex 플러그인은 Claude Code 안에서 코드 리뷰, 디버깅 위임, 백그라운드 작업 등을 수행할 수 있게 해주는데, 이 스킬은 **언제, 어떤 명령어를 써야 하는지** Claude가 스스로 판단할 수 있도록 도와줍니다.

### 주요 기능

- **의사결정 매트릭스**: Codex를 쓸지 말지, 어떤 명령어를 쓸지 자동 판단
- **프롬프팅 가이드**: Codex에게 좋은 요청을 보내기 위한 컨텍스트 수집/구성 가이드
- **자동 버전 체크**: Codex 플러그인 업데이트를 감지하고 가이드를 자동으로 최신 상태로 유지

### 자동 버전 체크

이 스킬은 하루 1회 Codex 플러그인의 최신 버전을 확인합니다:

1. GitHub 원격 최신 버전과 로컬 설치 버전을 비교 → 새 버전이 있으면 업데이트 안내
2. 설치된 플러그인 버전과 스킬의 가이드 버전을 비교 → 다르면 가이드 자동 업데이트

> **참고**: 버전 체크에 `gh` CLI가 필요합니다. 미설치 또는 네트워크 불가 시에도 스킬은 정상 작동하며, 버전 체크만 건너뜁니다.

`_meta.json`에서 설정을 변경할 수 있습니다:

| 필드 | 설명 | 기본값 |
|------|------|--------|
| `codex_plugin_version` | 현재 가이드가 기반하는 플러그인 버전 | 설치 시점 버전 |
| `last_checked` | 마지막 버전 체크 날짜 (YYYY-MM-DD) | 설치일 |
| `auto_update` | `true`: 자동 업데이트 / `false`: 매번 확인 | `true` |
| `source_repo` | 플러그인 원본 레포 | `openai/codex-plugin-cc` |

### 요구 사항

- [Claude Code](https://claude.ai/code) (플러그인 시스템 지원 버전)
- [Codex CLI](https://github.com/openai/codex) (`npm install -g @openai/codex`)
- ChatGPT 구독(무료 포함) 또는 OpenAI API 키
- (선택) [GitHub CLI](https://cli.github.com/) — 자동 버전 체크에 사용

### Codex 플러그인 설치

```bash
# Claude Code에서 실행
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Codex CLI 로그인:
```bash
!codex login
```

### 스킬 설치

`claude-code/skills/codex-usage-guide/` 폴더를 통째로 `~/.claude/skills/`에 복사하면 됩니다.

```bash
# 클론
git clone https://github.com/nebulakes-prog/shared-notes.git

# 복사
cp -r shared-notes/claude-code/skills/codex-usage-guide ~/.claude/skills/

# Claude Code 재시작하면 자동으로 스킬이 활성화됩니다
```

### 포함된 파일

```
codex-usage-guide/
├── SKILL.md                          # 의사결정 가이드 + 버전 자동 체크
├── _meta.json                        # 버전 추적 + 자동 업데이트 설정
└── references/
    ├── commands-reference.md         # 명령어 상세 옵션
    └── prompting-guide.md            # Codex에게 좋은 요청 보내는 법
```

### 참고

- Codex 플러그인 공식 레포: [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)
- Codex CLI 공식 레포: [openai/codex](https://github.com/openai/codex)

## License

MIT
