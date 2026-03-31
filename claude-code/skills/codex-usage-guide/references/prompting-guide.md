# Codex에게 좋은 요청 보내기

이 문서는 Claude가 Codex에 작업을 위임할 때, 요청 품질을 높이기 위해 참조하는 가이드.
XML 블록 조립은 `codex:gpt-5-4-prompting` internal skill이 자동 처리하므로,
이 문서는 그 전 단계 — **무엇을 담아서 보낼지** 판단하는 데 집중한다.

## Codex의 특성 (Claude와의 차이)

| 특성 | Codex | Claude |
|------|-------|--------|
| 컨텍스트 | 요청 시점에 전달받은 것만 앎 | 대화 전체 맥락 보유 |
| 도구 접근 | 로컬 파일시스템, git, 터미널 | 동일 + 대화형 사용자 상호작용 |
| 강점 | 단일 작업 집중, 도구 활용 끈기 | 멀티턴 대화, 전략적 판단 |
| 약점 | 대화 히스토리 없음, 사용자 의도 추론 어려움 | 장시간 단일 작업에 토큰 소모 큼 |

**핵심**: Codex는 내 대화 맥락을 모른다. 사용자와 나 사이에서 논의된 배경, 결정, 제약 조건을 Codex가 알 수 있도록 요청에 명시적으로 포함해야 한다.

## 요청 전 준비 체크리스트

Codex에 작업을 보내기 전에 다음을 확인:

### 1. 컨텍스트 수집
- [ ] 관련 파일 경로와 핵심 코드 위치
- [ ] 에러 메시지 또는 실패하는 테스트 명
- [ ] 사용자가 언급한 제약 조건 (성능, 호환성, 특정 라이브러리 등)
- [ ] 이전 시도에서 실패한 접근법 (Codex가 같은 실수를 반복하지 않도록)

### 2. 범위 한정
- [ ] 작업 하나만 포함 (여러 작업 = 여러 run)
- [ ] "완료"의 기준이 명확한가?
- [ ] 변경 범위가 제한되어 있는가? (무관한 리팩토링 방지)

### 3. 모드 결정
- [ ] 읽기만? → review / adversarial-review
- [ ] 쓰기 필요? → rescue (기본 --write)
- [ ] 오래 걸릴까? → --background
- [ ] 이전 작업 이어서? → --resume

## 좋은 요청 vs 나쁜 요청

### Bad: 맥락 없는 막연한 요청
```
/codex:rescue fix the bug
```
→ Codex가 뭘 고쳐야 하는지, 어디서 버그가 발생하는지 모름.

### Good: 구체적 컨텍스트 포함
```
/codex:rescue investigate why tests/auth.test.ts line 45 fails with 401.
The JWT middleware in src/middleware/auth.ts should accept tokens with
aud claim "api://default" but currently rejects them.
We already tried adding audience validation — that didn't work.
```
→ 파일 위치, 에러, 기대 동작, 실패한 시도가 모두 포함.

### Bad: 여러 작업 묶기
```
/codex:rescue review the API, fix the auth bug, and update the docs
```
→ 3개의 독립적 작업. 각각 별도 run으로.

### Good: 단일 작업
```
/codex:rescue fix the auth middleware JWT validation in src/middleware/auth.ts.
Expected: tokens with aud "api://default" pass validation.
Current: all tokens rejected with 401.
Verify fix by running: npm test -- --grep "JWT"
```
→ 하나의 명확한 작업 + 검증 방법.

### Bad: 사용자 결정을 Codex에 떠넘기기
```
/codex:rescue should we use Redis or Memcached for caching?
```
→ 아키텍처 결정은 Opus/Claude가 적합.

### Good: 구체적 조사 위임
```
/codex:rescue investigate the connection pooling behavior in src/db/pool.ts.
Check if connections are properly returned after timeout.
The pool exhaustion happens under load (>100 concurrent requests).
Look at the error pattern in logs: "pool exhausted after 30s".
```
→ 관찰된 증거 기반의 구체적 조사.

## 대화 맥락 전달 패턴

사용자와의 대화에서 Codex에 전달해야 할 정보가 있을 때:

```
/codex:rescue [작업 설명]

Background:
- [사용자가 언급한 제약 조건]
- [이전에 시도했지만 실패한 접근법]
- [관련 파일: path/to/file.ts]

Expected outcome:
- [구체적인 완료 기준]

Verify by:
- [테스트 명령 또는 확인 방법]
```

## 결과 처리 원칙

- Codex 출력은 그대로 사용자에게 전달 (요약/의역 금지)
- 리뷰 결과에서 이슈 발견 시 → 사용자에게 어떤 것을 수정할지 확인 후 진행
- Codex 실패 시 → Claude가 대신 답변하지 않음, 실패 사실만 보고
- Codex가 수정을 했다면 → 변경된 파일 목록 명시

## rescue 시 effort/model 선택 기준

| 상황 | model | effort |
|------|-------|--------|
| 빠른 확인, 단순 수정 | `spark` | 기본 |
| 일반적인 디버깅/구현 | 기본 (설정 따름) | 기본 |
| 복잡한 근본 원인 조사 | 기본 | `high` 또는 `xhigh` |
| 비용 절약이 중요한 루틴 | `gpt-5.4-mini` | `medium` |
