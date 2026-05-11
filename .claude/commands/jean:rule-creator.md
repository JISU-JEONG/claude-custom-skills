---
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
argument-hint: "{스캔 대상 경로 or 비워서 기본 스캔}"
description: 기존 스킬을 분석해 rule(`CLAUDE.md` 또는 `.claude/rules/*.md`)로 옮기면 좋은 부분을 찾아 제안·생성·정리한다. 사용자가 직접 호출. 기존 rule과 중복·모순 자동 검출. 스킬에서 추출하는 도구이므로 분석 대상 스킬이 먼저 존재해야 함. rule을 처음부터 만들고 싶을 때는 `~/.claude/CLAUDE.md` 또는 `.claude/rules/<topic>.md`를 직접 편집할 것.
---

# jean:rule-creator

기존 스킬·커맨드 안에 있는 "절차가 아닌 사실·컨벤션"을 추출해 별도 rule 파일로 옮기는 리팩터링 도구.

기본 응답 언어는 한국어. 사용자가 영어로 답하면 따라간다.

## 동작 흐름

5단계: **Discover → Analyze → Propose → Generate → Cleanup**

각 단계는 사용자 승인 후 다음으로 진행. 분석은 LLM 패턴 인식이므로 100% 결정적이지 않다 — 따라서 Step 3 의 사용자 승인이 핵심이다.

---

## Step 1. Discover

스캔 대상을 모두 로드한다.

<scan_targets>
**스킬·커맨드 (분석 대상)**:
- `./.claude/skills/**/SKILL.md`
- `./.claude/commands/*.md`
- `~/.claude/skills/**/SKILL.md`
- `~/.claude/commands/*.md`

**기존 rule (중복·모순 비교 대상)**:
- `./CLAUDE.md`, `./.claude/CLAUDE.md`, `./CLAUDE.local.md`
- `~/.claude/CLAUDE.md`
- `./.claude/rules/**/*.md`, `~/.claude/rules/**/*.md`

**제외**:
- `~/.claude/plugins/**` — read-only, 업데이트 시 덮어쓰임
</scan_targets>

사용자가 인자로 특정 경로를 줬다면 그 범위로만 한정. 기본은 위 전체 스캔.

스캔 결과를 사용자에게 짧게 요약 — "스킬 N개, 기존 rule 파일 M개 발견. 분석 들어갑니다."

---

## Step 2. Analyze

각 스킬 본문을 읽고 rule 후보를 분류한다.

### Rule 후보의 패턴

- 매번 동일하게 적용되는 사실 (예: "API 핸들러는 `src/api/handlers/`")
- 절차가 아닌 컨벤션·스타일 (예: "2-space 들여쓰기", "함수명 camelCase")
- 특정 paths 에만 해당되는 항상-적용 규칙 (예: "`src/api/**/*.ts` 파일은 입력 검증 필수")
- 도메인 지식·아키텍처 결정 (예: "이 코드베이스는 Repository 패턴을 사용한다")

### Rule 후보가 아닌 것

- 다단계 절차 — 스킬에 두는 게 맞음
- 사용자 호출 트리거가 있는 워크플로우
- 결정적이어야 할 자동 동작 — `/jean:hook-creator` 영역
- 한 번만 실행되는 작업

### 중복·모순 검출

후보를 기존 rule 과 항상 비교한다.

| 발견 | 처리 |
|---|---|
| 기존 rule 에 이미 있음 | 이관 불필요. "기존 rule 사용" 으로 표시 |
| 기존 rule 과 충돌 (예: tab vs space) | 사용자에게 "어느 쪽이 정답?" 묻기 |
| 기존 rule 과 보완 관계 | 통합 추천 (한 파일로 합침) |
| 신규 | 일반 추출 후보로 진행 |

### 후보 0개일 때

분석 결과 추출 가치 있는 후보가 하나도 없으면:

- 약한 후보를 억지로 끌어올리지 않는다 (false positive 방지)
- "스킬 N개 분석 결과, 추출 권장할 rule 후보 없습니다" 한 줄 보고 후 즉시 종료
- 보고에 분석 근거 짧게 포함 (예: "본문이 모두 절차·다단계 워크플로우로 구성됨", "기존 rule 에 이미 모두 반영됨")
- 사용자가 "그래도 약한 후보라도 보고 싶다" 라고 하면 그때 임계치 낮춰 재분석

---

## Step 3. Propose

분석 결과를 표로 사용자에게 제시한다.

<example>
| # | 출처 스킬 | 추출 후보 (요약) | 제안 위치 | 제안 paths | 중복/충돌 | 동작 |
|---|---|---|---|---|---|---|
| 1 | knovel-write | 회차 5,000~6,500자 등 한국어 문체 원칙 | `.claude/rules/korean-style.md` | `.claude/commands/knovel-write.md` (스킬 SKILL — invoke 시 자동 로드) | 없음 | 신규 |
| 2 | knovel-plan | Strand 비중 정의 (Quest 45%, Fire 20%, …) | `./CLAUDE.md` | (전역, 모든 세션) | 일부 중복 — 기존 CLAUDE.md 와 통합 | 통합 |
| 3 | data-agent | 신규 인물은 영문 ID 사용 (코드 내) | `.claude/rules/data-conventions.md` | `knovel/**/*.py` (소스 편집 시) | 없음 | 신규 |
| 4 | knovel-spicy | "수위 10 기본 허용" | (보류) | - | 기존 CLAUDE.md 와 충돌 가능 — 사용자 확인 필요 | 보류 |
</example>

각 항목에 사용자가 **수락 / 거절 / 수정** 표시 → 승인된 것만 Step 4 로.

`AskUserQuestion` 으로 일괄 또는 항목별 확인. 항목 많으면 Top 5 먼저 제시 후 점진 처리.

---

## Step 4. Generate

승인 항목별로 결정 트리를 따라 파일을 작성한다.

### 4-1. 스코프 선택

| 스코프 | 위치 | 사용 케이스 |
|---|---|---|
| User | `~/.claude/CLAUDE.md` 또는 `~/.claude/rules/*.md` | 본인 모든 프로젝트 |
| Project | `./CLAUDE.md`, `./.claude/CLAUDE.md`, `./.claude/rules/*.md` | 팀 공유 (커밋 대상) |
| Local | `./CLAUDE.local.md` 또는 추가 (gitignored) | 본인 + 이 프로젝트만 |

기본값: **Project**. 사용자가 명시하지 않으면 project 권장하되 항상 확인.

### 4-2. 파일 형태 선택

**Default: paths 적용**. "이 rule 이 *진짜로 모든 파일 / 모든 작업* 에 적용되는가?" 의문을 거치고 명확한 yes 일 때만 paths 생략한다. 도메인·언어·디렉토리·스킬 종속 rule 은 모두 paths 대상 — 이쪽이 다수다.

| 조건 | 선택 |
|---|---|
| 도메인·언어·디렉토리·스킬 특화 (대부분의 케이스) | `.claude/rules/<topic>.md` + `paths:` |
| 정말 모든 파일·작업에 적용 (드물다) | `.claude/rules/<topic>.md` (paths 없음) 또는 `CLAUDE.md` 섹션 |
| 짧고 단일 토픽 + 정말 전역 (5줄 이하) | `CLAUDE.md` 의 적절한 섹션 |

### 4-3. paths 작성 가이드

**paths 의 의미** — `paths:` 는 *Claude 가 해당 path 의 파일을 read 할 때* 이 rule 을 컨텍스트로 로드하라는 지시다. 즉 paths = **rule 의 트리거 파일** 이지 *작업 산출물 위치* 가 아니다.

**매칭 대상 결정 트리**:

| rule 적용 시점 | paths 매칭 대상 |
|---|---|
| 특정 스킬·커맨드 동작 중 적용 | 그 스킬의 `.claude/commands/<skill>.md` 또는 `.claude/skills/<skill>/SKILL.md` (스킬 invoke 시 본문 read → rule 자동 로드) |
| 특정 소스 코드 편집 중 적용 | 소스 파일 패턴 (예: `src/api/**/*.ts`) |
| 특정 도메인 텍스트·데이터 파일 편집 중 적용 | 도메인 파일 패턴 (예: `docs/api/**/*.md`) |

⚠️ **antipattern — 산출물 경로를 paths 로 등록**: 산출물 (스킬이 새로 *Write* 하는 파일) 은 보통 read 안 되므로 paths 매칭이 일어나지 않아 rule 이 로드되지 않는다. 산출물 작성 중 rule 이 적용되길 원하면 *그 산출물을 만드는 스킬의 SKILL.md 경로*를 paths 에 등록한다 — 스킬 본문이 read 되는 순간 rule 이 함께 컨텍스트로 들어와 LLM 이 그것을 따라 산출물을 작성한다.

**와일드카드 vs 명시적 listing**:

- **명시적 listing (default)** — 현재 존재하는 파일들을 정확히 나열. 신규 파일은 자동 제외 (수동 추가). 의도가 명확하다.
  ```yaml
  paths:
    - ".claude/commands/knovel-write.md"
    - ".claude/agents/writer.md"
  ```
- **와일드카드** — `**/*.ts` 등. 신규 파일 자동 포함이 *명시적 의도*일 때만 사용. 와일드카드 도입은 사용자에게 한 번 묻고 결정.
  ```yaml
  paths:
    - "src/api/**/*.ts"
  ```

**Thin shim 제외** — 다른 스킬을 호출만 하는 짧은 wrapper 커맨드는 paths 에서 제외한다. 이유: shim 본문 자체에는 rule 이 작동할 콘텐츠가 없고, 실제 rule 트리거 대상은 shim 이 위임하는 *진짜 스킬의 SKILL.md*. shim 까지 paths 에 넣으면 노이즈만 늘고 효과 없음.

**glob 검증 (와일드카드 사용 시)**:

```bash
python3 -c "import glob; print(len(glob.glob('src/api/**/*.ts', recursive=True)))"
```

매칭 0이면 사용자에게 경고 — 오타 또는 파일 부재.

### 4-4. 작성 원칙 (jean:skill-creator 와 동일)

- **Positive framing** — "do X" 우선. 부정 지시는 *이유* 동반
- **구체성** — "format code properly" → "use 2-space indentation"
- **검증 가능성** — 모호한 표현 지양
- **사이즈** — `CLAUDE.md` 200줄 이하, `.claude/rules/*.md` 토픽 단위 분리
- **markdown 구조** — 헤더·불릿·표 활용

### 4-5. 기존 rule 통합 (Propose 단계에서 "통합" 으로 표시된 항목)

기존 rule 파일을 Read → 추출 내용 자연스럽게 머지 → diff 사용자에게 보여주고 승인 → Edit 적용.

---

## Step 5. Cleanup

원본 스킬에서 이관된 부분을 제거한다. cross-reference 한 줄을 추가할지 여부는 **rule 의 paths 매칭 대상**에 따라 분기한다.

**Cross-reference 결정 트리**:

| rule 의 paths 매칭 대상 | cross-reference |
|---|---|
| 원본 스킬의 `SKILL.md` 자체 (스킬 invoke 시 SKILL 본문 read → rule 자동 컨텍스트 로드) | **생략** — redundant. 본문에서만 해당 부분 제거 |
| 산출물 또는 외부 파일 (스킬 invoke 시점에 rule 자동 로드 안 됨) | **유지** — LLM 이 작성 시점에 rule 발견하도록 한 줄 명시 |
| paths 없음 (CLAUDE.md 추가 또는 unconditional rule) | **생략** — 모든 세션에 항상 자동 로드되므로 redundant |

rule 의 cleanup 은 텍스트 이동뿐이라 즉시 안전하다. (Hook 의 cleanup 은 검증 후 — `/jean:hook-creator` 와의 차이.)

<example>
**Case A — paths 가 SKILL.md 매칭 (cross-ref 생략)**:

```yaml
# .claude/rules/korean-style.md
paths:
  - ".claude/commands/knovel-write.md"
```

스킬 본문 변화 — 해당 섹션 *통째로 제거*. cross-ref 안 넣음. 스킬이 invoke 될 때 paths 매칭으로 rule 이 자동 로드되므로 SKILL.md 안에 다시 언급할 필요 없음.

```diff
- ## 한국어 문체 원칙
- - 회차 5,000~6,500자 / 문단 1~4줄
- - 5단계 존대 레지스터
- - 호칭 진화 (intimacy 0~5)
```
</example>

<example>
**Case B — paths 가 외부 도메인 파일 매칭 (cross-ref 유지)**:

```yaml
# .claude/rules/api-doc-conventions.md
paths:
  - "docs/api/**/*.md"
```

스킬은 `docs/api/*.md` 산출물을 생성. 스킬 invoke 시점엔 rule 자동 로드 안 되므로 cross-ref 한 줄 유지:

```diff
- ## API 문서 작성 가이드
- [12줄 컨벤션 본문]
+ ## API 문서 작성
+ → `.claude/rules/api-doc-conventions.md` 참고
+   (paths: `docs/api/**/*.md` — 산출물 작성 시점에 자동 로드)
```
</example>

cleanup 적용 전 항상 diff 를 사용자에게 보여주고 승인 받는다.

이관 후 스킬 본문이 현저히 짧아지면 사용자에게 "이 스킬은 추가 분리 가치 있음 — 다음 라운드에서 더 추출 가능" 만 보고하고 추가 cleanup 은 다음 호출로 넘긴다.

---

## 작성 best practice (skill-creator 와 동일 적용)

스킬 작성 시와 동일한 4가지 체크:

1. **Positive framing** — "X 하지 마" 보다 "Y 해라"
2. **XML 태그** — 인자·예시·맥락 분리
3. **Few-shot 3원칙** — 예시는 *relevant + diverse + structured*, 3~5개
4. **Colleague test** — 새 동료가 이 rule 만 보고 행동 가능한가?

자세한 가이드는 `~/.claude/commands/jean:skill-creator.md` 의 "Skill Writing Guide" 참고.

---

## 안전장치

- 모든 파일 작성·수정은 **diff 미리 보여주고 사용자 승인 후** 적용
- 기존 rule 파일 수정 전 `.bak` 또는 git stash 안내
- 중복·충돌 케이스는 자동 진행 금지 — 반드시 사용자 결정 받기
- 스코프 선택은 항상 명시적으로 확인 (특히 user vs project)

---

## 다른 진입점에서 들어왔을 때

| 사용자 의도 | 안내 |
|---|---|
| "rule 처음부터 만들고 싶어" | 이 도구는 기존 스킬 분석 전용. `~/.claude/CLAUDE.md` 또는 `.claude/rules/<topic>.md` 직접 편집 권장 |
| 결정적 자동 동작 (포맷·차단·세션 주입) | `/jean:hook-creator` 사용 권장 |
| 새 스킬 작성 | `/jean:skill-creator` 사용 권장 |

---

## 산출물

- rule 파일 1개 이상 (신규 또는 기존 통합)
- 원본 스킬 cleanup diff
- 작업 요약: 추출 N개 / 통합 M개 / 보류 K개

---

## 공식 문서 기반

이 도구의 모든 결정 트리는 다음 공식 문서를 기반으로 한다:

- [Memory / CLAUDE.md](https://code.claude.com/docs/en/memory) — 계층, paths, @import, 200줄 가이드
- [Settings](https://code.claude.com/docs/en/settings) — `claudeMdExcludes`, 스코프 우선순위
- [Skills](https://code.claude.com/docs/en/skills) — 스킬 본문이 어떤 내용을 포함해야 하는지의 기준
