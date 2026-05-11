---
name: jean:skill-sync
description: |
  skills/agents/rules 파일 수정·추가 후 일관성 자동 점검·정리 스킬.
  포맷 준수, 스킬 간 상호작용 계약, rules paths 불일치, 잔제 패턴, 유령 참조, 중복 내용을 탐지하고 수정한다.
  "/jean:skill-sync" 단독 실행 또는 jean:skill 완료 후 자동 호출.
  인자로 파일명을 넘기면 해당 파일 관련 범위만 스캔.
---

# jean:skill-sync — 프리미티브 일관성 검사 및 정리

.claude/ 하위 skills·agents·rules 파일 간 일관성을 점검하고 발견된 문제를 수정한다.

## 입력

```
/jean:skill-sync                   # 전체 스캔
/jean:skill-sync novel-drafter     # 특정 파일명 관련 범위만 스캔
```

## 1단계: 파일 목록 수집

<instructions>
아래 명령으로 대상 파일 목록을 구성한다.

```bash
echo "=== SKILLS ===" && find .claude/skills -name "SKILL.md" 2>/dev/null | sort
echo "=== AGENTS ===" && find .claude/agents -name "*.md" 2>/dev/null | sort
echo "=== RULES ===" && find .claude/rules -name "*.md" 2>/dev/null | sort
```

인자가 있으면 해당 파일명을 포함하는 파일과 그 파일을 참조하는 rules만 대상으로 좁힌다.

각 파일을 Read해 내용을 파악한다. 이후 모든 단계 분석은 이 내용을 기반으로 한다.
</instructions>

## 2단계: 포맷 준수 검사

<instructions>
각 파일이 프리미티브 유형별 필수 포맷을 갖추고 있는지 확인한다.

```bash
python3 - <<'PYEOF'
import os, re, glob

issues = []

# --- SKILL 검사 ---
for p in sorted(glob.glob(".claude/skills/*/SKILL.md")):
    content = open(p).read()
    name = os.path.basename(os.path.dirname(p))

    # frontmatter 필수 항목
    m = re.search(r'^---\n(.*?)\n---', content, re.DOTALL)
    if m:
        fm = m.group(1)
        for field in ['name:', 'description:']:
            if field not in fm:
                issues.append(f"[SKILL 포맷] {name}/SKILL.md: frontmatter '{field}' 없음")
    else:
        issues.append(f"[SKILL 포맷] {name}/SKILL.md: frontmatter 없음")

    # 필수 구조 태그
    if '<instructions>' not in content:
        issues.append(f"[SKILL 포맷] {name}/SKILL.md: <instructions> 태그 없음")
    if '<output_format>' not in content:
        issues.append(f"[SKILL 포맷] {name}/SKILL.md: <output_format> 태그 없음")

# --- AGENT 검사 ---
for p in sorted(glob.glob(".claude/agents/*.md")):
    content = open(p).read()
    name = os.path.splitext(os.path.basename(p))[0]

    # frontmatter 필수 항목
    m = re.search(r'^---\n(.*?)\n---', content, re.DOTALL)
    if m:
        fm = m.group(1)
        for field in ['name:', 'description:', 'model:', 'tools:']:
            if field not in fm:
                issues.append(f"[AGENT 포맷] {name}.md: frontmatter '{field}' 없음")
    else:
        issues.append(f"[AGENT 포맷] {name}.md: frontmatter 없음")

    # 필수 구조 태그
    if '<output_format>' not in content:
        issues.append(f"[AGENT 포맷] {name}.md: <output_format> 반환 형식 없음")

if issues:
    for i in issues:
        print(i)
else:
    print("OK: 포맷 전체 준수")
PYEOF
```
</instructions>

## 3단계: 스킬 간 상호작용 계약 검사

<instructions>
오케스트레이터 스킬이 에이전트를 호출하는 프롬프트 형식과, 해당 에이전트의 입력 프로토콜이 일치하는지 LLM 판단으로 분석한다.

**절차:**

1. 각 skill 파일에서 "Task 도구로 {agent명} 에이전트를 호출" 패턴 또는 드래프터 프롬프트 템플릿(`<drafter_prompt_template>` 등)을 찾는다.

2. 해당 에이전트 파일의 "입력 프로토콜" 또는 "## 입력" 섹션을 확인한다.

3. 아래 항목을 대조한다:
   - 오케스트레이터가 넘기는 **필드명**이 에이전트 입력 프로토콜의 필드명과 일치하는가?
   - 오케스트레이터가 넘기는 **섹션 헤더**(`## 플롯`, `## 세계관` 등)가 에이전트가 기대하는 헤더와 일치하는가?
   - 오케스트레이터가 **필수 필드를 누락**하지 않았는가?
   - 에이전트의 **반환 형식**(`<output_format>`)을 오케스트레이터가 올바르게 사용하는가?

**판정 기준:**
- 필드명·헤더 불일치 → 계약 위반
- 에이전트가 기대하는 필드를 오케스트레이터가 전달하지 않음 → 누락
- 오케스트레이터가 에이전트 반환값을 잘못된 형식으로 파싱 → 파싱 불일치

발견된 불일치는 "어느 파일의 어느 부분이 왜 다른지" 함께 기록한다.
</instructions>

## 4단계: Rules paths 유효성 검사

<instructions>
각 rules 파일의 `paths:` 항목이 실제 존재하는 파일을 가리키는지 확인한다.

```bash
python3 - <<'PYEOF'
import os, re

rules_dir = ".claude/rules"
if not os.path.isdir(rules_dir):
    print("rules 디렉토리 없음")
    exit()

issues = []
for fname in sorted(os.listdir(rules_dir)):
    if not fname.endswith(".md"):
        continue
    fpath = os.path.join(rules_dir, fname)
    content = open(fpath).read()
    m = re.search(r'^---\n(.*?)\n---', content, re.DOTALL)
    if not m:
        continue
    paths = re.findall(r'^\s*-\s*"([^"]+)"', m.group(1), re.MULTILINE)
    for p in paths:
        if not os.path.exists(p):
            issues.append(f"[누락] {fname}: '{p}'")

if issues:
    for i in issues:
        print(i)
else:
    print("OK: paths 전체 유효")
PYEOF
```
</instructions>

## 5단계: 잔제 패턴 스캔

<instructions>
모든 대상 파일에서 아래 잔제 패턴을 탐지한다.

```bash
# 버전 역사 기술
grep -rn "v1에서는\|이전 버전에서는\|원래는.*지만 이제\|과거에.*방식\|구 버전" \
  .claude/skills .claude/agents .claude/rules 2>/dev/null | head -20

# deprecated 주석
grep -rn "deprecated\|더 이상 사용\|폐기됨" \
  .claude/skills .claude/agents .claude/rules 2>/dev/null | head -20
```

각 발견 건: 파일명·줄번호·내용 기록.
</instructions>

## 6단계: 유령 참조 탐지

<instructions>
삭제된 파일 이름이 다른 파일에 남아 있는지 확인한다.

```bash
python3 - <<'PYEOF'
import os, re, glob

existing = set()
for p in glob.glob(".claude/agents/*.md"):
    existing.add(os.path.splitext(os.path.basename(p))[0])
for p in glob.glob(".claude/skills/*/SKILL.md"):
    existing.add(os.path.basename(os.path.dirname(p)))

issues = []
skip = {"skill", "rule", "hook", "agent", "idea", "sync"}
for p in glob.glob(".claude/**/*.md", recursive=True):
    content = open(p).read()
    refs = set(re.findall(r'\bnovel-[a-z-]+', content))
    for ref in refs:
        if ref not in existing and ref not in skip:
            issues.append(f"[유령] {p}: '{ref}'")

for i in sorted(set(issues))[:20]:
    print(i)
if not issues:
    print("OK: 유령 참조 없음")
PYEOF
```
</instructions>

## 7단계: Rules-대상 정합성 분석

<instructions>
각 rule의 내용과 paths 대상이 적합한지 LLM 판단으로 분석한다.

**판단 기준:**

| rule 내용 유형 | 적합한 대상 | 부적합한 대상 |
|---|---|---|
| 산문 집필 스타일·어휘 | 집필 에이전트(drafter), 자연화 에이전트(naturalizer) | 오케스트레이터 스킬, 평가 에이전트 |
| 평가·검증 기준 | 평가 에이전트(critic), 검증 스킬 | 집필 에이전트 |
| 데이터 모델·파일 구조 | 파일 I/O 담당 skills/agents 전체 | 특정 집필 에이전트만 |
| 장르 톤·어휘 금지선 | 모든 출력 생성 파일 | — |
| 플롯 분량 기준 | 플롯 작성 스킬, 집필 에이전트 | 자연화·평가 에이전트 |

1단계에서 Read한 rule 내용과 paths를 비교해 불일치를 기록한다.
불일치 발견 시: "이유와 권고 수정 방향" 함께 기록.
</instructions>

## 8단계: 중복 내용 탐지

<instructions>
rules에서 이미 관리되는 내용이 skill/agent 본문에 재기술됐는지 확인한다.

탐지 방법:
- 각 rule의 핵심 고유 표현(40자 이상)을 skill/agent 파일 본문에서 검색
- 발견 시: "rule {파일}로 이미 관리되는 내용이 {대상}에 중복 기술됨" 으로 기록

```bash
grep -rn "의학 완곡어\|직접 표기\|expansion ratio" .claude/skills .claude/agents 2>/dev/null | head -10
```

중복 탐지는 skill/agent의 중복 부분만 삭제하도록 권고한다. rule 이동 불필요.
</instructions>

## 9단계: 결과 보고 및 수정

<instructions>
발견된 이슈를 분류별로 보고한다. 이슈가 없으면 "일관성 검사 통과" 보고 후 종료.

이슈가 있으면:

**자동 수정 대상** (안전한 삭제·경로 제거):
- 존재하지 않는 paths 항목 삭제
- 잔제 패턴 줄 삭제
- 유령 참조 제거

자동 수정 대상은 변경 diff를 모두 보여주고 `AskUserQuestion`으로 일괄 승인받은 뒤 Edit한다.

**판단 필요 대상** (포맷 위반, 계약 불일치, rules-대상 불일치, 중복):
각 항목별로 권고안과 이유를 제시하고 `AskUserQuestion`으로 개별 승인받아 수정한다.

수정 완료 후 4단계를 재실행해 paths 불일치 0건 확인.
</instructions>

<output_format>
## skill-sync 완료

**스캔 범위**: {전체 / {파일명} 기준}
**검사**: Skills {N}개 · Agents {N}개 · Rules {N}개

| 검사 항목 | 발견 | 수정 |
|---|---|---|
| 포맷 준수 (frontmatter·태그) | {N}건 | {N}건 |
| 스킬 간 계약 불일치 | {N}건 | {N}건 |
| Rules paths 유효성 | {N}건 | {N}건 |
| 잔제 패턴 | {N}건 | {N}건 |
| 유령 참조 | {N}건 | {N}건 |
| Rules-대상 불일치 | {N}건 | {N}건 |
| 중복 내용 | {N}건 | {N}건 |
</output_format>
