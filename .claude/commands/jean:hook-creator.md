---
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
argument-hint: "{스캔 대상 경로 or 비워서 기본 스캔}"
description: 기존 스킬을 분석해 hook(`settings.json` 또는 `SKILL.md` frontmatter)으로 옮기면 좋은 결정적 동작을 찾아 제안·생성·검증·정리한다. 사용자가 직접 호출. 기존 hook과 중복·모순 자동 검출. dry-run 검증과 단계적 설치(local-first)로 안전성 확보. 스킬에서 추출하는 도구이므로 분석 대상 스킬이 먼저 존재해야 함. hook을 처음부터 만들고 싶을 때는 `.claude/settings.json`을 직접 편집할 것.
---

# jean:hook-creator

기존 스킬·커맨드 안에 있는 "결정적이어야 할 자동 동작"을 추출해 hook 으로 옮기는 리팩터링 도구. dry-run 검증과 단계적 설치로 안전성을 확보한다.

기본 응답 언어는 한국어. 사용자가 영어로 답하면 따라간다.

## 동작 흐름

8단계: **Discover → Analyze → Propose → Generate → Dry-run → Local install → Verify → Promote+Cleanup**

각 단계는 사용자 승인 후 다음으로. **Step 5·6·7 은 어떤 경우에도 skip 하지 않는다** (안전성의 핵심).

---

## Step 1. Discover

<scan_targets>
**스킬·커맨드 (분석 대상)**:
- `./.claude/skills/**/SKILL.md`
- `./.claude/commands/*.md`
- `~/.claude/skills/**/SKILL.md`
- `~/.claude/commands/*.md`

**기존 hook (중복·모순 비교 대상)**:
- `./.claude/settings.json` 의 `hooks` 필드
- `./.claude/settings.local.json` 의 `hooks` 필드
- `~/.claude/settings.json` 의 `hooks` 필드
- 각 SKILL.md frontmatter 의 `hooks:` 필드

**제외**:
- `~/.claude/plugins/**` — read-only, 업데이트 시 덮어쓰임
</scan_targets>

사용자에게 짧게 요약 — "스킬 N개, 기존 hook M개 발견. 분석 들어갑니다."

---

## Step 2. Analyze

각 스킬 본문에서 hook 후보 패턴을 식별한다.

### Hook 후보의 패턴

| 표현 | 매핑되는 이벤트 |
|---|---|
| "X 후에 항상 Y 해라" | `PostToolUse` |
| "X 전에 항상 Y 체크해라" | `PreToolUse` |
| "세션 시작 시 항상 ... 로드해라" | `SessionStart` |
| "사용자 입력에서 X 가 보이면 ..." | `UserPromptSubmit` |
| "응답 종료 시 ... 알려라" | `Stop` / `Notification` |
| "컴팩트 전 ... 백업해라" | `PreCompact` |
| "파일 X 가 바뀌면 ..." | `FileChanged` |

추가 신호:
- LLM 판단에 맡기지 말고 *결정적*이어야 하는 동작 (보안 체크, 포맷팅, 차단)
- 매번 같은 부작용을 발생시켜야 하는 작업
- 사용자가 까먹지 말아야 할 자동 처리

### Hook 후보가 아닌 것

- 사용자 판단·창의성 필요한 작업
- 컨텍스트 의존적 의사결정
- 다단계 절차 — 스킬에 둠
- 항상 적용되는 사실 — `/jean:rule-creator` 영역

### 중복·모순 검출

| 발견 | 처리 |
|---|---|
| 동일 이벤트 + 동일 matcher 의 hook 이 이미 있음 | 통합 추천 또는 보류 |
| 두 PostToolUse 가 같은 파일을 수정 (충돌) | 사용자에게 우선순위 묻기 |
| 차단 hook 이 기존 차단 hook 과 겹침 | 통합 |
| 신규 | 일반 추출 후보로 진행 |

### 후보 0개일 때

분석 결과 추출 가치 있는 후보가 하나도 없으면:

- 약한 후보를 억지로 끌어올리지 않는다 (false positive 방지)
- "스킬 N개 분석 결과, 추출 권장할 hook 후보 없습니다" 한 줄 보고 후 즉시 종료
- 보고에 분석 근거 짧게 포함 (예: "결정적 자동 동작 패턴 발견되지 않음", "기존 hook 에 이미 모두 반영됨")
- 사용자가 "그래도 약한 후보라도 보고 싶다" 라고 하면 그때 임계치 낮춰 재분석

---

## Step 3. Propose

분석 결과를 표로 사용자에게 제시한다.

<example>
| # | 출처 스킬 | 추출 후보 | 이벤트 | matcher | type | 정의 위치 | 차단 |
|---|---|---|---|---|---|---|---|
| 1 | commit | "커밋 전 npm run format" | `PostToolUse` | `Edit\|Write` | command | `.claude/settings.json` | 정보형 |
| 2 | deploy | "금요일 배포 금지" | `PreToolUse` | `Bash` (`if: "Bash(*deploy*)"`) | command | `.claude/settings.json` | 차단 |
| 3 | knovel-write | "시작 시 state.json 의 current_step 로드" | `SessionStart` | `startup\|resume` | command | SKILL.md frontmatter | 정보형 |
</example>

각 항목 사용자 승인 → Step 4 로.

---

## Step 4. Generate

승인 항목별로 결정 트리를 따라 핸들러 + 설정을 만든다.

### 4-1. 이벤트 선택 가이드

| 의도 | 이벤트 | matcher 예시 |
|---|---|---|
| 편집 후 자동 처리 (포맷·테스트) | `PostToolUse` | `Edit\|Write` |
| 위험 명령 차단 | `PreToolUse` | `Bash` + `if: "Bash(rm *)"` |
| 세션 시작 시 컨텍스트 주입 | `SessionStart` | `startup\|resume` |
| 사용자 프롬프트 검사·차단 | `UserPromptSubmit` | (matcher 없음) |
| 응답 종료 후 알림 | `Stop` 또는 `Notification` | (matcher 없음) |
| 컴팩트 전 후처리 | `PreCompact` | `manual\|auto` |
| 환경변수 동기화 | `CwdChanged` / `FileChanged` | 파일명 |

### 4-2. 핸들러 type 선택

| type | 사용 케이스 |
|---|---|
| `command` | 셸 스크립트 (90% 케이스) |
| `http` | 외부 endpoint 로 POST |
| `mcp_tool` | MCP 툴 호출 |
| `prompt` | LLM 판단형 (의사결정 필요) |
| `agent` | 서브에이전트 위임 |

### 4-3. 정의 위치 결정 트리 ⚠️ 중요

| 동작 범위 | 위치 |
|---|---|
| 스킬 활성 동안만 동작 | **SKILL.md frontmatter `hooks:`** (스킬 본체에 잔류 — cleanup 시 본문에서만 빼고 frontmatter 로 이동) |
| 항상 동작 + 팀 공유 | `./.claude/settings.json` |
| 항상 동작 + 본인만 | `./.claude/settings.local.json` |
| 모든 프로젝트 | `~/.claude/settings.json` |

> Step 6 에서 **무조건 `.claude/settings.local.json`** 부터 설치 후, Step 7 검증 통과 시 Step 8 에서 위 표의 최종 위치로 승격.

### 4-4. 핸들러 스크립트 작성

`.claude/hooks/<name>.sh` 에 분리 (settings.json 인라인은 가독성 ↓).

```bash
chmod +x .claude/hooks/<name>.sh
```

**환경변수**:
- `$CLAUDE_PROJECT_DIR` — 프로젝트 루트
- `$CLAUDE_PLUGIN_ROOT` — 플러그인 디렉토리 (해당시)
- `$CLAUDE_PLUGIN_DATA` — 플러그인 영구 데이터

**stdin** 으로 hook input JSON 받음 — `jq` 로 파싱.

**exit code 컨벤션**:

| 코드 | 의미 | 비고 |
|---|---|---|
| `0` | 정상. stdout JSON 으로 결정 제어 가능 | `permissionDecision`, `additionalContext`, `updatedInput` |
| `2` | blocking error | 차단 가능 이벤트만 (PreToolUse·UserPromptSubmit·Stop 등) |
| 기타 | 비차단 경고 | stderr 가 transcript 에 표시 |

<example>
**예시 — 금요일 deploy 차단**:

`.claude/hooks/block-friday-deploy.sh`:
```bash
#!/bin/bash
COMMAND=$(jq -r '.tool_input.command')
if [[ "$COMMAND" == *deploy* ]] && [[ $(date +%u) == "5" ]]; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "금요일 배포 금지"
    }
  }'
fi
exit 0
```

`.claude/settings.local.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block-friday-deploy.sh"
          }
        ]
      }
    ]
  }
}
```
</example>

<example>
**예시 — 편집 후 자동 포맷**:

`.claude/hooks/auto-format.sh`:
```bash
#!/bin/bash
FILE_PATH=$(jq -r '.tool_input.file_path')
if [[ "$FILE_PATH" == *.ts || "$FILE_PATH" == *.tsx ]]; then
  npx prettier --write "$FILE_PATH" 2>/dev/null
fi
exit 0
```

settings 패치:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/auto-format.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```
</example>

<example>
**예시 — SessionStart 컨텍스트 주입 (SKILL.md frontmatter)**:

`SKILL.md`:
```yaml
---
name: knovel-write
description: ...
hooks:
  SessionStart:
    - matcher: "startup|resume"
      hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/hooks/load-state.sh"
---
```

`.claude/hooks/load-state.sh`:
```bash
#!/bin/bash
CURRENT_STEP=$(jq -r '.current_step.stage // "unknown"' .knovel/state.json 2>/dev/null)
jq -n --arg step "$CURRENT_STEP" '{
  hookSpecificOutput: {
    hookEventName: "SessionStart",
    additionalContext: ("현재 단계: " + $step)
  }
}'
exit 0
```
</example>

---

## Step 5. Dry-run 테스트 ⚠️ 필수

핸들러가 시뮬레이션 input 으로 의도대로 동작하는지 검증한다.

### 테스트 케이스 작성 (최소 2개)

- **trigger 케이스**: 차단/처리되어야 할 input
- **무관 케이스**: 통과해야 할 input

### 실행

```bash
# trigger 케이스
echo '{"tool_name":"Bash","tool_input":{"command":"npm run deploy"}}' \
  | bash .claude/hooks/block-friday-deploy.sh
echo "exit=$?"

# 무관 케이스
echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' \
  | bash .claude/hooks/block-friday-deploy.sh
echo "exit=$?"
```

### 검증 항목

- exit code (의도한 값과 일치하는가)
- stdout JSON 스키마 (필요 필드 존재, `jq .` 로 파싱 가능)
- stderr 메시지 (사용자에게 의미 있는가)
- 60초 타임아웃 안전

### 실패 시

사용자에게 보고 → 핸들러 수정 → 재시도. 이 단계 통과 전에는 Step 6 으로 가지 않는다.

### 테스트 픽스처 보존

`.claude/hooks/_test/<name>.test.sh` 에 dry-run 스크립트를 남겨둔다 — 다음 수정 시 회귀 검증용.

---

## Step 6. Local install

검증 통과한 hook 을 **`.claude/settings.local.json`** 에 설치한다 (gitignore 대상이라 팀 영향 없음).

기존 `settings.local.json` 이 있으면 Read → 머지 → 사용자에게 diff 보여주고 승인 → Edit.

설치 후 사용자에게 안내:

```
설치 완료. 검증을 위해:
1. 새 Claude Code 세션을 띄워주세요 (`claude`)
2. `/hooks` 명령으로 hook 등록 확인
3. 의도한 트리거 명령 시도 → 예상 동작: <차단/처리 내용>
4. 무관 명령 시도 → 정상 통과 확인
5. 결과를 알려주세요
```

---

## Step 7. Verify

사용자가 실제 세션에서 hook 동작을 검증한 결과를 받는다.

### 통과 기준

- trigger 케이스: 의도한 차단·처리 발생
- 무관 케이스: 영향 없음
- `/hooks` 메뉴에 정상 표시
- 사용자가 "OK" 또는 동등 의사 표시

### 실패 시

- 핸들러 수정 → Step 5 부터 재실행
- 또는 hook 제거 (`.claude/settings.local.json` 에서 삭제) → 사용자에게 종료 보고

---

## Step 8. Promote + Cleanup

검증 통과 후에만 진행. 이 순서를 지키지 않으면 hook 이 작동 안 하는 상태로 스킬에서만 제거되는 사고가 난다.

### 8-1. Promote

`.claude/settings.local.json` 의 hook 설정을 Step 4-3 에서 결정한 최종 위치로 이동.

- `.claude/settings.json` 으로 옮기는 경우: local 에서 제거 + 최종 위치에 추가 (사용자 승인 후 git add 안내)
- `~/.claude/settings.json` 으로 옮기는 경우: local 에서 제거 + 사용자 settings 에 추가
- SKILL.md frontmatter 로 옮기는 경우: local 에서 제거 + frontmatter `hooks:` 추가
- 그대로 local 유지: 별도 작업 없음

각 경로별 머지 diff 를 사용자에게 보여주고 승인 후 적용.

### 8-2. Cleanup

원본 스킬에서 이관된 지시를 제거 + cross-reference 한 줄로 대체.

<example>
**Before** (스킬 본문):
```markdown
## 배포 절차
1. 테스트 실행
2. **금요일에는 배포 금지** ← 이관 대상
3. 빌드
4. 푸시
```

**After**:
```markdown
## 배포 절차
1. 테스트 실행
2. (금요일 차단은 PreToolUse hook 으로 자동 처리 — `.claude/hooks/block-friday-deploy.sh`)
3. 빌드
4. 푸시
```
</example>

cleanup diff 도 사용자 승인 후 적용.

---

## 안전성 핵심 원칙

<safety_rules>
**우회 금지 항목**:

1. **Step 5 (dry-run)** — 어떤 경우에도 skip 하지 않는다. 사용자가 "빨리 해줘" 라고 해도 dry-run 은 거친다
2. **Step 6 (local-first)** — `.claude/settings.local.json` 검증 통과 전에 `.claude/settings.json` 또는 `~/.claude/settings.json` 에 직접 쓰지 않는다
3. **Step 7 → 8 순서** — 검증 통과 전에는 원본 스킬을 수정하지 않는다 (검증 실패 시 기능이 사라지는 사고 방지)

**롤백 가이드 동봉**:

모든 작업 결과 보고 끝에 다음을 안내:

- hook 단일 비활성: 해당 hook entry 를 settings 에서 삭제 (또는 주석)
- 전역 비활성: settings 에 `"disableAllHooks": true` 추가
- 스킬 복원: cleanup 이전 git diff 로 복원
</safety_rules>

---

## 다른 진입점에서 들어왔을 때

| 사용자 의도 | 안내 |
|---|---|
| "hook 처음부터 만들고 싶어" | 이 도구는 기존 스킬 분석 전용. `.claude/settings.json` 의 `hooks` 필드 직접 편집 권장 |
| 항상 적용되는 사실·컨벤션 | `/jean:rule-creator` 사용 권장 |
| 새 스킬 작성 | `/jean:skill-creator` 사용 권장 |

---

## 산출물

- 핸들러 스크립트 `.claude/hooks/<name>.sh` (`chmod +x` 적용)
- 테스트 픽스처 `.claude/hooks/_test/<name>.test.sh`
- settings 패치 (최종 위치)
- 원본 스킬 cleanup diff
- 작업 요약: 추출 N개 / dry-run 결과 / verify 결과 / promote 위치별 분포

---

## 공식 문서 기반

이 도구의 모든 결정 트리·스키마·exit code 컨벤션은 다음 공식 문서를 기반으로 한다:

- [Hooks reference](https://code.claude.com/docs/en/hooks) — 18+ 이벤트, 5종 핸들러, exit code, JSON 스키마
- [Hooks guide](https://code.claude.com/docs/en/hooks-guide) — 실용 패턴, `/hooks` 메뉴
- [Settings](https://code.claude.com/docs/en/settings) — `hooks` 필드, `disableAllHooks`, 스코프 우선순위
- [Skills](https://code.claude.com/docs/en/skills) — SKILL.md frontmatter `hooks:` 필드
