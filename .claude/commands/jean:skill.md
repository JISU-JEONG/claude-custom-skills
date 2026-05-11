---
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
argument-hint: "{만들고 싶은 기능/행동 설명 or 리팩터링할 스킬 경로}"
description: 사용자 요구사항을 받아 jean:skill-creator / jean:rule-creator / jean:hook-creator / jean:claude-md-creator / jean:agent-creator 중 필요한 것을 선택·순서대로 실행하는 통합 오케스트레이터. 새 기능·자동화·행동을 처음부터 만들거나 기존 스킬을 올바른 프리미티브(스킬/룰/훅/CLAUDE.md/에이전트)로 리팩터링할 때 사용. 스킬 생성 시 eval pass_rate ≥ 0.95 달성까지 자동 루프. 모든 수정 시 사이드 이펙트 항상 검사. "만들어줘", "추가해줘", "이 스킬 정리해줘", "자동화해줘", "skill/rule/hook 만들어", "CLAUDE.md 만들어줘", "프로젝트 컨텍스트 설정해줘", "에이전트 만들어줘", "서브에이전트 설계해줘", "전문가 위임 구조", "어떤 프리미티브에 넣어야 해" 등 Claude Code 행동 정의가 필요한 모든 상황에서 가장 먼저 사용할 것.
---

# jean:skill

사용자 요구사항을 분석해 **jean:skill-creator**, **jean:rule-creator**, **jean:hook-creator**, **jean:claude-md-creator**, **jean:agent-creator** 중 필요한 도구를 선택하고 올바른 순서로 실행하는 오케스트레이터.

기본 응답 언어는 한국어. 사용자가 영어로 답하면 따라간다.

---

## 다섯 도구의 역할 분담

<context>
Claude Code 행동은 다섯 종류의 프리미티브로 구성된다:

| 프리미티브 | 담당 도구 | 핵심 기준 |
|---|---|---|
| **CLAUDE.md** | jean:claude-md-creator | 매 세션 자동 로드되는 프로젝트 컨텍스트 (스택, 명령어, 아키텍처, 주의사항) |
| **Skill** | jean:skill-creator | 다단계 절차, 사용자가 명시 호출 (`/jean:xxx`) |
| **Agent** | jean:agent-creator | 오케스트레이터가 위임하는 격리 전문가 (모델·도구 권한·입출력 계약 설계) |
| **Rule** | jean:rule-creator | 항상 적용되는 사실·컨벤션·스타일, paths 매칭으로 자동 로드 |
| **Hook** | jean:hook-creator | 이벤트 기반 결정적 자동 실행 (포맷, 차단, 세션 주입) |

다섯 종류는 서로 배타적이지 않다. 하나의 요구사항이 여럿에 걸칠 수 있다.
</context>

---

## Step 1. 인터뷰 · 분류

인자 또는 첫 메시지에서 요구사항을 파악하고 어떤 프리미티브가 필요한지 결정한다.

### 분류 기준

<instructions>
각 요구사항 요소에 대해 아래 질문을 순서대로 적용한다:

```
이 내용이:

├─ 매 세션 Claude가 자동으로 알고 있어야 할 프로젝트 컨텍스트인가?
│   (스택, 명령어, 아키텍처, 팀 규칙, 주의사항 등)
│   → CLAUDE.md 필요 (jean:claude-md-creator)
│
├─ 사용자가 명시적으로 호출하는 다단계 워크플로우인가?
│   → Skill 필요 (jean:skill-creator)
│
├─ 오케스트레이터가 복잡한 작업을 격리된 전문가에게 위임해야 하는가?
│   (병렬 처리, 컨텍스트 격리, 모델·도구 세분화 제어 필요)
│   → Agent 필요 (jean:agent-creator)
│
├─ 어떤 컨텍스트에서든 항상 자동으로 적용되어야 할 사실·컨벤션인가?
│   → Rule 필요 (jean:rule-creator)
│
├─ 특정 이벤트가 발생하면 결정적으로 자동 실행되어야 하는 동작인가?
│   → Hook 필요 (jean:hook-creator)
│
└─ 위 중 여러 가지가 섞여있는가?
    → 해당하는 모든 프리미티브 필요, Step 2 실행 순서대로 처리
```

요구사항이 불분명하면 한 번 물어보되, 인자로 충분한 정보가 왔으면 바로 분류 결과를 제안한다.
분류 결과를 사용자에게 보여주고 진행 승인을 받은 뒤 Step 2로 넘어간다.
</instructions>

<examples>
<example>
요구사항: "Python 파일 저장할 때마다 자동으로 ruff 포맷팅"
분류:
- "저장할 때마다 자동" = 이벤트 기반, 결정적 → Hook (PostToolUse)
- 사용자 명시 호출 없음 → Skill 불필요
결론: Hook만
</example>

<example>
요구사항: "한국어 소설 쓸 때 문체 가이드를 항상 따르게"
분류:
- "항상 따르게" = 스킬 invoke 시 자동 로드 컨벤션 → Rule (paths: 소설 스킬 SKILL.md)
- 다단계 절차 없음 → Skill 불필요
결론: Rule만
</example>

<example>
요구사항: "git blame 스타일로 파일 히스토리를 요약해주는 기능"
분류:
- 사용자가 명시 호출, 다단계 분석 워크플로우 → Skill
- 자동 실행 아님 → Hook 불필요
- 항상 적용 규칙 없음 → Rule 불필요
결론: Skill만 (eval 95% 루프 적용)
</example>

<example>
요구사항: "커밋 전 1) lint 자동 체크 2) 커밋 메시지 규칙 3) 전체 플로우 수동 실행 기능"
분류:
- "lint 자동 체크" = 이벤트 기반 결정적 → Hook (PreToolUse)
- "커밋 메시지 규칙" = 항상 적용 컨벤션 → Rule
- "전체 플로우 수동 실행" = 다단계 워크플로우, 사용자 호출 → Skill
결론: Skill + Rule + Hook 전부
</example>
</examples>

---

## Step 2. 실행 순서

<instructions>
결정된 프리미티브 조합에 따라 아래 순서로 실행한다.

```
CLAUDE.md만:                    claude-md-creator
CLAUDE.md + Skill:              claude-md-creator → skill-creator
CLAUDE.md + Skill + Rule:       claude-md-creator → skill-creator → rule-creator
CLAUDE.md + Skill + Rule + Hook: claude-md-creator → skill-creator → rule-creator → hook-creator
Skill + Agent:                  skill-creator → agent-creator
Skill + Agent + Rule:           skill-creator → agent-creator → rule-creator
Skill + Agent + Hook:           skill-creator → agent-creator → hook-creator
Skill + Rule + Hook:            skill-creator → rule-creator → hook-creator
Skill + Rule:                   skill-creator → rule-creator
Skill + Hook:                   skill-creator → hook-creator
Rule + Hook:                    rule-creator → hook-creator
Skill만:                        skill-creator
Agent만:                        agent-creator
Rule만:                         rule-creator
Hook만:                         hook-creator
```

**CLAUDE.md를 먼저 만드는 이유**: 프로젝트 컨텍스트가 확정돼야 이후 Skill·Rule·Hook이 어떤 스택과 컨벤션 기반으로 작동할지 결정할 수 있다.

**Skill을 CLAUDE.md 다음에 만드는 이유**: 스킬 본문이 있어야 rule-creator와 hook-creator가 "어디서 무엇을 추출할지" 파악할 수 있다.

**Agent를 Skill 다음에 만드는 이유**: 오케스트레이터 스킬이 존재해야 에이전트의 입출력 계약과 호출 컨텍스트를 정확히 설계할 수 있다.

**기존 스킬 리팩터링**: skill-creator 단계를 건너뛰고 rule-creator → hook-creator 순으로 진행한다.

각 단계는 사용자 승인 후 다음으로 넘어간다. 중간에 방향이 바뀌면 남은 경로를 재계산한다.
</instructions>

---

## Step 3. 각 도구 실행

각 도구의 상세 지침은 해당 커맨드 파일에 있다. 실행 시점에 해당 파일을 Read하고 그 지침을 따른다.

### 3-A. skill-creator 실행

`~/.claude/commands/jean:skill-creator.md` 를 Read하고 지침을 따른다.

<instructions>
**95% eval 루프 규칙 (이 오케스트레이터의 핵심 제약):**

스킬 초안 완성 후 jean:skill-creator의 eval 루프를 실행한다.
루프 종료 조건: `grading.json`의 `summary.pass_rate ≥ 0.95`

- jean:skill-creator 본래 종료 조건("사용자가 만족하면 종료")을 이 기준으로 재정의한다.
- 95%에 도달하기 전에는 rule-creator나 hook-creator로 넘어가지 않는다.
- 5회 이상 반복해도 95%에 도달하지 못하면 사용자에게 현황 보고 후 계속 여부를 묻는다.
- eval pass_rate 계산: `passed / total` (grading.json summary 기준)

각 루프 반복 후 현재 점수를 사용자에게 보고한다: "현재 X% (목표 95%)"

**스킬 완성 후 동반 도구 제안 (필수):**

95% 달성 후, 초기 분류에 Rule·Hook·CLAUDE.md가 없었더라도 반드시 아래 질문을 한다:

> "스킬이 완성됐습니다. 추가로 실행할 도구를 선택해 주세요:
> - **rule-creator**: 이 스킬에서 항상 적용될 컨벤션·사실을 자동 로드 규칙으로 추출
> - **hook-creator**: 이 스킬과 연동할 이벤트 자동화(포맷, 차단 등) 검사
> - **claude-md-creator**: 프로젝트 CLAUDE.md에 이 스킬 관련 컨텍스트 추가
> - 건너뛰기: 스킬만으로 충분함"

사용자가 선택한 도구를 Step 2 실행 순서에 추가해 이어서 실행한다. "건너뛰기"를 선택하면 Step 4로 넘어간다.
</instructions>

### 3-B. rule-creator 실행

`~/.claude/commands/jean:rule-creator.md` 를 Read하고 지침을 따른다.

스캔 범위: 3-A에서 생성된 스킬 파일, 또는 사용자가 지정한 기존 스킬.
완료 조건: rule 파일 생성 완료 또는 "후보 없음" 보고.

### 3-C. hook-creator 실행

`~/.claude/commands/jean:hook-creator.md` 를 Read하고 지침을 따른다.

스캔 범위: rule-creator 실행 후의 스킬 상태 (rule이 이미 추출된 후).

<instructions>
hook-creator의 **dry-run → local install → verify → promote** 단계는 이 오케스트레이터가 생략시킬 수 없다. 사용자가 "빨리 해줘"라고 해도 이 단계들은 반드시 거친다. 이유: dry-run을 건너뛰면 잘못된 hook이 조용히 설치되어 예상치 못한 동작이 발생하고, local-first를 건너뛰면 팀 전체 설정에 검증 안 된 hook이 들어갈 수 있다.
</instructions>

완료 조건: hook 검증 완료 또는 "후보 없음" 보고.

### 3-E. claude-md-creator 실행

`~/.claude/commands/jean:claude-md-creator.md` 를 Read하고 지침을 따른다.

실행 시점: Step 2 실행 순서에서 CLAUDE.md가 포함된 경우 가장 먼저 실행한다.

<instructions>
**스코프 선택 확인 필수**: claude-md-creator는 항상 사용자에게 저장 위치를 확인한다. 오케스트레이터는 이 단계를 생략시킬 수 없다. 스코프 오선택은 민감한 정보를 팀 전체에 노출하거나 팀 설정을 개인 설정으로 덮어쓸 위험이 있다.

스코프 선택지:
- `CLAUDE.md` (프로젝트 루트) — 팀 공유, git 추적
- `.claude/CLAUDE.md` — 프로젝트 로컬, git 추적
- `CLAUDE.local.md` — 개인용, .gitignore 권장
- `~/.claude/CLAUDE.md` — 전역 (모든 프로젝트에 적용)

**300줄 예산**: 생성된 CLAUDE.md가 300줄을 초과하면 claude-md-creator가 경고하고 슬림화를 제안한다.

**업데이트 모드**: 기존 CLAUDE.md가 있으면 전체 재작성이 아닌 diff/추가 형태로 변경을 제안한다.
</instructions>

완료 조건: CLAUDE.md 생성 또는 업데이트 완료, 스코프 확인 완료.

### 3-F. agent-creator 실행

`~/.claude/commands/jean:agent-creator.md` 를 Read하고 지침을 따른다.

실행 시점: Step 2 실행 순서에서 Agent가 포함된 경우. Skill이 함께 있을 때는 skill-creator 완료 후 실행한다.

<instructions>
**설계 인터뷰 필수**: agent-creator는 모델 선택·도구 권한·입출력 계약의 세 가지 설계 결정을 사용자와 함께 결정한다. 오케스트레이터는 이 과정을 건너뛸 수 없다. 이유: 세 가지 결정이 잘못되면 에이전트가 과도한 권한을 갖거나, 오케스트레이터와 계약 불일치가 발생한다.

**파일 위치 확인**: 전역(~/.claude/agents/)과 프로젝트(.claude/agents/) 중 어디에 저장할지 사용자에게 확인한다.

**Skill과 함께 만드는 경우**: 오케스트레이터 스킬이 완성된 후 에이전트 입출력 계약을 설계하면, 실제 호출 컨텍스트를 기반으로 더 정확한 프로토콜을 만들 수 있다.
</instructions>

완료 조건: 에이전트 파일 생성 완료, 모델·도구·계약 설계 확인 완료.

### 3-D. 수정 후 잔제 검사 (모든 스킬 수정에 공통 적용)

skill-creator·rule-creator·hook-creator 중 어느 도구를 써서 스킬 파일을 수정했든, 수정 직후 아래 잔제 패턴을 스캔하고 발견 시 제거한다. 잔제가 쌓이면 스킬이 점점 길어지고 미래 실행자가 현행 규칙과 과거 주석을 구분하지 못하는 문제가 생긴다.

<instructions>
**잔제 패턴 및 처리 방법:**

| 패턴 | 예시 | 처리 |
|------|------|------|
| 버전 역사 기술 | "v1에서는 N+1이었는데..." / "이전 버전에서는 X를 했으나" | 삭제 |
| 구 버전 금지 + 역사 설명 | "X하지 말것 (과거에 Y 방식을 쓰다 문제가 생겼기 때문)" | "X 대신 Z를 써라"로만 남김 |
| 이유 없는 부정 금지 | "절대 X하지 마세요" | 긍정 표현으로 전환: "Y를 해라" |
| deprecated 주석 | "// deprecated" / "— 더 이상 사용하지 않음" | 해당 섹션 전체 삭제 |
| 과거 방식 비교 | "원래는 X였지만 이제 Y가 맞다" | "Y를 써라"만 남기고 비교 부분 삭제 |
| 존재하지 않는 참조 | 삭제된 파일·도구·API 경로 참조 | 참조 제거 또는 현행 경로로 교체 |

**판단 기준:** 해당 텍스트를 삭제해도 스킬의 현재 동작에 영향이 없으면 삭제한다.

**제외 조건:** 각 스킬의 prompting 규칙(positive framing, XML tags, few-shot 3원칙, colleague test)을 위반하게 되는 경우 수정하지 않는다. 변경 전 diff를 사용자에게 보여주고 승인받는다.
</instructions>

---

## Step 4. 사이드 이펙트 확인

<instructions>
모든 파일 생성·수정 **전후**에 아래 체크를 실행한다. 이 검사를 건너뛰면 기존 동작이 조용히 깨질 수 있다. 영향받는 요소가 발견되면 사용자에게 알리고 함께 처리 방법을 결정한다.

### 스킬 파일 수정 시

```bash
# 이 스킬을 paths로 참조하는 rule 파일 검색
SKILL_BASENAME=$(basename <skill-path>)
grep -rl "$SKILL_BASENAME" ~/.claude/rules/ ./.claude/rules/ ~/.claude/CLAUDE.md ./CLAUDE.md 2>/dev/null

# 이 스킬을 참조하는 hook 검색 (settings.json)
grep -rl "$SKILL_BASENAME" ~/.claude/settings.json ./.claude/settings.json ./.claude/settings.local.json 2>/dev/null
```

확인 항목:
- 이 스킬 파일을 paths로 매칭하는 rule이 있는가? (스킬 이름 변경 시 paths 갱신 필요)
- 이 스킬 이름을 참조하는 hook이 있는가?
- 이 스킬이 다른 스킬을 @import하거나 invoke하는가?

### rule 파일 수정 시

```bash
RULE_BASENAME=$(basename <rule-path>)
grep -rl "$RULE_BASENAME" ~/.claude/ ./.claude/ 2>/dev/null
```

확인 항목:
- 이 rule을 @import하거나 참조하는 다른 파일이 있는가?
- paths 패턴이 변경됐다면 기존에 커버하던 파일이 여전히 매칭되는가?
  ```bash
  python3 -c "import glob; print(len(glob.glob('<new-pattern>', recursive=True)))"
  ```

### hook 스크립트 수정 시

```bash
HOOK_BASENAME=$(basename <hook-script>)
grep -rl "$HOOK_BASENAME" ~/.claude/settings.json ./.claude/settings.json ./.claude/settings.local.json 2>/dev/null
```

확인 항목:
- settings.json에서 이 스크립트를 참조하는 모든 hook entry가 여전히 유효한가?
- 동일 이벤트+matcher를 가진 다른 hook과 충돌하지 않는가?
- 수정 후 dry-run을 재실행해 동작이 의도대로인지 재확인한다.

### settings.json 수정 시

확인 항목:
- 기존 hook entry가 머지 과정에서 누락되지 않았는가? (Read → merge → diff → 승인 순서 준수)
- `disableAllHooks`가 실수로 추가되지 않았는가?
</instructions>

---

## Step 5. skill-sync 실행

<instructions>
모든 도구 실행이 완료된 후 `/jean:skill-sync`를 실행한다.

수정·생성된 파일이 있으면 해당 파일명을 인자로 전달한다:
```
/jean:skill-sync {수정된 파일명}
```

skill-sync 완료 보고를 받은 뒤 Step 6으로 넘어간다.
</instructions>

## Step 6. 최종 요약

모든 도구 실행 완료 후 아래 형식으로 요약한다:

<output_format>
## 작업 완료 요약

### 생성된 프리미티브
| 종류 | 경로 | 설명 |
|------|------|------|
| CLAUDE.md | `CLAUDE.md` 또는 `~/.claude/CLAUDE.md` | ... |
| Skill | `~/.claude/commands/...` | ... |
| Agent | `~/.claude/agents/...` 또는 `.claude/agents/...` | ... |
| Rule | `~/.claude/rules/...` | ... |
| Hook | `.claude/hooks/...` | ... |

### Eval 결과 (Skill 생성 시)
- 최종 pass_rate: **X%** (목표: 95%)
- 반복 횟수: N회

### 사이드 이펙트 검사
- 영향받은 파일: [목록 또는 "없음"]
- 처리 내용: [조치 사항 또는 "해당 없음"]

### 다음 권장 작업 (있을 경우)
- ...
</output_format>

---

## 조율 원칙

**분석 중복 없음**: 각 도구가 자체 스캔을 수행한다. 오케스트레이터는 경로와 컨텍스트만 전달하고 재분석하지 않는다.

**부분 실행 허용**: 사용자가 "rule만", "hook 빼고"라고 하면 해당 도구만 실행한다. 강제하지 않는다.

**안전장치 위임 불가**: hook-creator의 dry-run/local-first/verify와 rule-creator의 사용자 승인 단계는 이 오케스트레이터가 bypass할 수 없다.

**95% 규칙 준수**: skill-creator의 eval 루프는 pass_rate ≥ 0.95 달성 전까지 계속된다. 이 기준은 오케스트레이터 고유의 제약으로, jean:skill-creator 본래의 "사용자 만족 시 종료" 조건을 덮어쓴다.

**컨텍스트 보존**: 도구 간 핸드오프 시점에 현재 상태(어떤 파일이 어디에 생겼는지, 무엇이 추출됐는지)를 명확히 기록해 다음 도구에 전달한다.
