---
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion
argument-hint: "{에이전트 역할/이름 설명, 비워두면 대화로 진행}"
description: Claude Code 에이전트(.claude/agents/<name>.md)를 설계·작성·검증한다. Skill과 달리 에이전트는 오케스트레이터가 위임하는 격리된 전문가. 모델 선택·도구 권한·입출력 계약 설계에 집중. eval 루프로 품질 검증. "에이전트 만들어줘", "서브에이전트 설계해줘", "전문가 위임 구조 만들어줘", "agent 추가해줘", "코드 리뷰 전담 에이전트", "병렬 처리용 에이전트", "격리 컨텍스트 필요해" 등 Claude Code 에이전트 파일이 필요한 상황에서 먼저 호출할 것.
---

# jean:agent-creator

에이전트(`.claude/agents/`)는 스킬과 근본적으로 다르다: **오케스트레이터(메인 Claude)가 `Agent` 도구를 통해 위임**하는 격리된 전문가다. 사용자가 `/command`로 직접 호출하지 않는다.

이 스킬이 집중하는 세 가지 설계 결정:
1. **모델** — haiku / sonnet / opus 중 적합한 것
2. **도구 권한** — 최소 권한 원칙으로 필요한 것만
3. **입출력 계약** — 오케스트레이터가 무엇을 넘기고, 에이전트가 무엇을 돌려주는가

> **eval 인프라 공유**: `~/.claude/skills/jean-skill-creator/` (grader, analyzer, aggregate_benchmark.py 재사용)

기본 응답 언어는 한국어. 사용자가 영어로 답하면 따라간다.

---

## 단계 1. 설계 인터뷰

인자 또는 대화에서 아래 다섯 가지를 파악한다. 인자로 충분하면 바로 제안하고 확인만 받는다.

<instructions>
**질문 순서 (인자에서 이미 알 수 있는 것은 건너뛴다):**

1. **전문 영역**: 이 에이전트가 수행할 한 가지 명확한 작업은?
   - 분석/읽기 전용인가, 파일 수정을 하는가, 외부 호출을 하는가?

2. **호출 관계**: 어떤 스킬/오케스트레이터가 이 에이전트를 호출하는가?
   - 루프에서 반복 호출되는가, 단발 위임인가?
   - 병렬로 여러 인스턴스가 동시에 실행되는가?

3. **입력 형식**: 오케스트레이터가 전달하는 프롬프트에 무엇이 담기는가?
   - 파일 경로? 데이터? 이전 결과? 반복 회차(N)?

4. **반환 형식**: 에이전트가 돌려줄 결과의 정확한 구조는?
   - 마크다운 리포트? JSON? 파일 직접 수정 후 요약?

5. **격리 요건**: 워크트리 격리(`isolation: "worktree"`)가 필요한가?
   - 실험적 코드 변경이라 메인 브랜치를 오염시키면 안 되는 경우에 필요

파악이 안 된 항목만 질문한다. 모두 확인되면 Step 2로 넘어간다.
</instructions>

<examples>
<example>
인자: "PR 코드 리뷰 전담 에이전트"
추론:
- 전문 영역: PR diff 읽고 품질/보안/성능 리뷰
- 호출: 코드 리뷰 스킬이 단발 위임
- 입력: PR 번호 또는 diff 경로
- 반환: 섹션별 마크다운 리포트
- 격리: 불필요 (읽기 전용)
→ 바로 설계 제안
</example>

<example>
인자: "DB 스키마 분석 에이전트, 여러 개 병렬로 돌릴 거야"
추론:
- 전문 영역: DB 스키마 파일 읽고 구조 분석
- 호출: 병렬 다중 인스턴스
- 입력: 각 스키마 파일 경로
- 반환: 구조화된 분석 결과 (파일당 독립)
- 격리: 불필요 (읽기 전용 + 병렬 안전)
→ 바로 설계 제안
</example>
</examples>

---

## 단계 2. 설계 결정

인터뷰 결과를 바탕으로 세 가지 핵심 설계를 결정하고 사용자에게 제안한다.

### 2-A. 모델 선택

<instructions>
아래 기준을 적용한다:

| 작업 특성 | 모델 |
|---|---|
| 단순 분류, 포맷 변환, 빠른 응답 | `haiku` |
| 일반 코드 작성, 분석, 리뷰 | `sonnet` (기본값) |
| 복잡한 설계 판단, 긴 컨텍스트 추론, 다단계 분석 | `opus` |

하나의 기준이 올 때 특성이 여럿 섞이면 상위 모델을 선택한다. 비용 민감 루프(반복 수십 회 이상)면 haiku/sonnet 선호를 사용자에게 알린다.
</instructions>

### 2-B. 도구 권한 설계

<instructions>
최소 권한 원칙: 에이전트가 실제로 사용할 도구만 허용한다. 권한이 좁을수록 사고 범위가 제한되어 안전하다.

| 에이전트 유형 | 권장 도구 |
|---|---|
| 읽기 전용 분석 | `Read, Glob, Grep` |
| 코드 수정 | `Read, Edit, Glob, Grep` |
| 파일 생성 포함 | `Read, Write, Edit, Glob, Grep` |
| 명령 실행 필요 | `Read, Bash` (최소한으로) |
| 외부 리서치 | `Read, WebSearch, WebFetch` |
| 전체 워크플로우 오케스트레이터 | `Read, Write, Edit, Bash, Agent` |

Write가 필요한지 신중히 판단한다: Edit(기존 파일 수정)만으로 충분하면 Write(새 파일 생성)는 제외한다.
</instructions>

### 2-C. 입출력 계약

오케스트레이터가 이 에이전트를 호출할 때의 프롬프트 구조와, 에이전트가 반환할 결과 구조를 정의한다. 계약이 명확할수록 오케스트레이터와 에이전트 사이의 통합 오류가 줄어든다.

---

## 단계 3. 에이전트 파일 작성

### 파일 위치

```
~/.claude/agents/<name>.md          # 전역 에이전트 (모든 프로젝트에서 사용 가능)
{project}/.claude/agents/<name>.md  # 프로젝트 에이전트 (이 프로젝트에서만)
```

전역으로 쓸 범용 에이전트면 `~/.claude/agents/`에, 특정 프로젝트 전용이면 프로젝트 `.claude/agents/`에 저장한다. 사용자에게 확인한다.

### 파일 포맷

```markdown
---
name: agent-name-kebab-case
description: |
  이 에이전트가 하는 일 한 줄.
  오케스트레이터가 언제 이 에이전트를 호출해야 하는지.
model: sonnet
tools: Read, Edit, Glob, Grep
---

# 에이전트 이름

## 역할

전문 영역 한 단락.

## 입력 프로토콜

오케스트레이터가 전달하는 프롬프트 형식 (구조화된 예시 포함).

## 처리 절차

1. 단계 1
2. 단계 2

## 반환 형식

반환할 결과의 정확한 구조 (템플릿 또는 예시 포함).

## 제약

이 에이전트가 해서는 안 되는 것 (이유 포함).
```

### 작성 원칙

**description이 오케스트레이터의 위임 판단 근거다.** 오케스트레이터는 description을 보고 "이 에이전트에 위임해야 하는가"를 결정한다. "언제 호출해야 하는가"를 명시할 것.

**입력 프로토콜은 예시로 고정한다.** 오케스트레이터가 보낼 프롬프트의 구조를 실제 예시로 보여주면 통합 오류가 크게 줄어든다:

```markdown
## 입력 프로토콜

오케스트레이터가 아래 형식으로 프롬프트를 구성한다:

```
[스킬 경로] ~/.claude/skills/my-skill/SKILL.md
[대상 파일] src/api/handler.py
[반복 회차] 2
```
```

**반환 형식은 파싱 가능하게 설계한다.** 오케스트레이터가 에이전트 결과를 읽고 다음 단계를 결정해야 한다면, 반환 포맷을 구조화하여 "패스 여부", "핵심 발견", "다음 조치" 등을 명확히 분리한다.

**루프에서 호출되는 에이전트는 회귀를 막는다.** 이전 iteration 결과와 비교할 수 있도록 입력에 이전 결과 경로를 포함하는 설계를 권장한다.

**금지 사항을 이유와 함께 명시한다.** "파일 X 수정 금지"가 아닌 "파일 X는 오케스트레이터가 관리하므로 이 에이전트는 읽기만 한다"처럼 이유를 붙이면 에이전트가 경계를 더 잘 지킨다.

---

## 단계 4. Eval 및 검증

에이전트 eval의 핵심 차이: **eval 프롬프트 = 오케스트레이터가 구성할 위임 메시지**다. 사용자가 직접 입력하는 것이 아니라 오케스트레이터가 보낼 내용을 시뮬레이션한다.

### Eval 전략

<instructions>
**with_agent / without_agent 비교:**
- `with_agent`: 작성한 에이전트 파일이 `~/.claude/agents/`에 설치된 상태에서 오케스트레이터 위임 프롬프트 실행
- `without_agent`: 동일 프롬프트를 에이전트 파일 없이 일반 Claude에게 실행 (baseline)

**평가 항목 (assertions):**
1. **계약 준수**: 반환 형식이 정의된 구조를 따르는가?
2. **범위 준수**: 허가되지 않은 파일을 수정하거나 제약을 넘지 않는가?
3. **태스크 완료**: 위임받은 작업의 핵심 결과물이 있는가?
4. **도구 절약**: 선언한 도구 이상을 사용하지 않는가?

**eval 프롬프트 작성 원칙:**
- "오케스트레이터가 실제로 구성할 프롬프트"를 그대로 사용한다
- 파일 경로, iteration 번호, 컨텍스트 데이터를 포함한다
- 에이전트가 처리할 실제 입력 데이터(fixture)를 준비한다
</instructions>

### Eval 실행 방법

```
워크스페이스: ~/.claude/jean-agent-creator-workspace/
구조:
  evals/evals.json              — eval 케이스 목록
  iteration-<N>/
    <eval-name>/
      eval_metadata.json
      with_agent/outputs/       — result.md, grading.json, timing.json
      without_agent/outputs/
    benchmark.json
    benchmark.md
```

**with_agent 서브에이전트 프롬프트 예시:**
```
에이전트 파일: ~/.claude/agents/<name>.md
태스크: <오케스트레이터 위임 메시지>
입력 데이터: <fixture 파일 경로 또는 내용>
출력 저장: <workspace>/iteration-N/<eval-name>/with_agent/outputs/result.md
```

**without_agent는** 동일 위임 메시지를 에이전트 파일 없이 일반 Claude에 그대로 넘긴다.

### 채점 및 집계

jean:skill-creator와 동일한 인프라를 재사용한다:

```bash
# 채점: ~/.claude/skills/jean-skill-creator/agents/grader.md 활용
# 집계:
cd ~/.claude/skills/jean-skill-creator && \
  python -m scripts.aggregate_benchmark \
    ~/.claude/jean-agent-creator-workspace/iteration-N \
    --skill-name jean:agent-creator
```

**95% 임계값**: jean:skill 오케스트레이터에서 호출된 경우 pass_rate ≥ 0.95 달성 전까지 반복한다.

각 루프 후 결과 보고: "현재 X% (목표 95%) — <주요 실패 패턴 1줄>"

---

## 단계 5. jean:skill 연동 업데이트

새 에이전트를 기존 스킬과 연결하는 경우, 해당 스킬 파일에 에이전트 호출 코드를 추가하고 jean:skill의 Step 4(사이드 이펙트 확인)를 실행한다.

---

## 완료 기준

아래 세 가지가 충족되면 완료:
1. `~/.claude/agents/<name>.md` (또는 프로젝트 경로) 생성
2. eval pass_rate ≥ 0.95 (jean:skill에서 호출된 경우) 또는 사용자가 직접 호출 시 검증 확인
3. 기존 스킬과 연결된 경우 사이드 이펙트 검사 완료
