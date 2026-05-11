---
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, AskUserQuestion
argument-hint: "{분석 주제 — 자유 형식}"
description: 코드베이스 심층 분석. 사용자가 지정한 주제·이슈에 대해 코드 전수 조사 후 파일:라인 단위 상세 리포트 생성. 검증 루프(최대 3회)로 false positive 제거. "어디서 발생하는지", "찾아줘", "누락된 곳", "감사", "이슈 추적" 같은 자유 주제 질문에 호출. 주제 고정 없음 — N+1 쿼리, 중복 코드, 에러 핸들링, dead code, 보안 취약점, 성능 핫스팟 등 무엇이든. 키워드 — 심층 분석, 코드 감사, 이슈 추적, 어디서, 찾아줘, deep analysis, code audit, hotspot, vulnerability scan.
---

# /jean:deep-analyze

사용자의 자유 주제 질문 → 코드 전수 조사 → 검증 루프 → 상세 결과 문서.

**스킬이 정의하는 건 *프로세스와 출력 포맷*이다. 주제는 사용자가 결정한다.** 어떤 프로젝트에서든 동작하며 스택을 자동 감지한다.

## 입력

`$ARGUMENTS` — 분석 주제 (자유 형식).

<examples>
<example>"N+1 쿼리 어디서 발생해?"</example>
<example>"중복 코드 찾아줘"</example>
<example>"에러 핸들링 빠진 곳"</example>
<example>"리팩터링 필요한 곳"</example>
<example>"매직 넘버 하드코딩된 곳"</example>
<example>"타입 힌트 없는 public 함수"</example>
<example>"인증 데코레이터 누락 엔드포인트"</example>
</examples>

주제가 모호하거나 빈 인자로 호출되면 `AskUserQuestion` 으로 명확화 요청 — 추측으로 진행하지 않는다.

---

## 핵심 원칙

<principles>
1. **코드 기반 진실** — 주장은 모두 Glob/Grep/Read 로 직접 확인한 것만 기재. 확인 불가 시 "[확인 필요]" 마커로 표시.
2. **정확한 수치 + 파일 목록** — 수량 주장 시 해당 파일 목록을 함께 나열. Grep 매치 수를 그대로 쓰지 않고 실제 해당 건만 필터링.
3. **시스템적 맥락 우선** — 상위 보호장치 (글로벌 핸들러·미들웨어·기본 세션 관리) 가 이슈를 커버하면 심각도 하향.
4. **기존 해결책 활용** — 수정 제안 작성 전 프로젝트 내 기존 유틸·헬퍼·repository 메서드 Grep. 이미 있으면 그것을 활용하는 방향으로 제안.
5. **백업 후 덮어쓰기** — 기존 결과 파일이 있으면 `docs/analysis/deep/_backup_YYYYMMDD-HHMMSS/` 로 옮긴 뒤 재생성 (사용자 작업 손실 방지).
6. **민감 정보 보호** — `.env` 등의 값은 변수명만 기재.
7. **생성물 디렉토리 제외** — `node_modules/`, `__pycache__/`, `.git/`, `.venv/`, `dist/`, `build/`, `target/`, `.gradle/`, `vendor/`, `.next/`, `out/` 는 분석 대상에서 빼고 시작.
</principles>

---

## Phase 0: 프로젝트 Discovery

스택 자동 감지. `/jean:analyze` 와 동일 로직.

| 감지 파일 | 스택 |
|-----------|------|
| `pyproject.toml` / `requirements.txt` / `setup.py` | Python |
| `package.json` | Node.js / TypeScript |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `pom.xml` / `build.gradle*` | Java / Kotlin |

프레임워크 세부 감지: import 패턴 또는 dependency 확인 → Flask / FastAPI / Django / Spring Boot / Express / NestJS / Gin 등.

기존 `docs/analysis/_INDEX.md` 또는 `docs/analysis/01-architecture.md` 가 있으면 핫스팟 디렉토리 우선순위로 활용 (선택적).

---

## Phase 1: 조사

### Step 1 — 주제 추출

`$ARGUMENTS` 에서 분석 주제를 추출한다.

<intent_handling>
- 인자 비어있음 또는 한 단어만 ("분석") → `AskUserQuestion` 으로 구체적 주제 묻기
- 인자 명확함 ("N+1 쿼리 어디?") → 그대로 진행
- 인자 모호함 ("개선 필요한 곳") → `AskUserQuestion` 으로 어느 측면 (성능·보안·구조 등) 인지 묻기
</intent_handling>

### Step 2 — 탐지 패턴 결정

주제 + 감지 스택에 맞는 Grep/Read 패턴을 결정한다. 패턴은 스킬이 고정하지 않고 Claude 가 주제를 이해해 적절히 정한다.

<example_patterns>
| 주제 | 탐지 관점 (언어 무관) |
|------|---------------------|
| N+1 쿼리 | loop 안에서 매 iteration 마다 DB 쿼리, lazy-loaded relationship 접근 |
| 중복 코드 | 10줄 이상 동일·유사 코드 블록, 같은 로직이 여러 파일에 반복 |
| 에러 핸들링 누락 | 포괄적 catch-all, 빈 catch, 외부 호출 시 예외 처리 없음 |
| dead code | 미호출 함수·메서드, 미사용 import, deprecated 마킹 후 참조 중 |
| 보안 | 인증 데코레이터·미들웨어 누락 엔드포인트, 문자열 보간 SQL, 하드코딩 시크릿 |
| 성능 | 전체 로딩 후 앱 레벨 필터링, 불필요한 SELECT *, 캐시 미적용 반복 쿼리 |
</example_patterns>

표는 시드용. 위 목록에 제한되지 않는다.

### Step 3 — 전수 조사

`Agent(Explore)` 를 투입해 코드 스캔. 프로젝트 규모에 따라 에이전트 수 조절:

| 규모 | 에이전트 |
|------|---------|
| 소규모 (~200 파일) | 단일 에이전트 |
| 중규모 (~1000 파일) | 디렉토리별 2~4개 병렬 |
| 대규모 (1000+ 파일) | 레이어별 분할 병렬 |

발견 후 **연쇄 탐색**:
- 발견 항목이 있는 파일의 *나머지 함수*에서 동일 패턴 확인
- 발견 항목이 있는 디렉토리의 *형제 파일* Grep
- dead code / orphan 판정 시 → 해당 모듈을 import 하는 파일 역추적, 1단계 깊이까지 unreachable 확인

### Step 4 — 초안 생성

발견 항목을 출력 템플릿에 맞춰 초안 문서로 정리한다 (출력 템플릿은 본문 하단 참조).

수정 제안 작성 시:
- 해당 함수 **전체** Read → 모든 분기 (if/elif/else, status별) 확인 후 제안
- 프로젝트 내 기존 유틸 Grep → 이미 해결책이 있으면 활용 방향으로 제안
- 수정 난이도는 기존 코드 활용 가능성을 반영해 평가

각 발견 항목 필수 필드:

| 필드 | 설명 |
|------|------|
| 파일:라인 | 정확한 위치 |
| 문제 코드 | 해당 코드 스니펫 (±5줄) |
| 왜 문제인가 | 원인 설명 |
| 심각도 | Critical / Warning / Info |
| 수정 제안 | 구체적 방법 또는 코드 예시 |

### 심각도 정의

<severity>
- **Critical** — 보호장치 없이 데이터 손실·보안 취약점 발생 가능. *글로벌 핸들러가 커버하면 Critical 자격 없음*.
- **Warning** — 글로벌 핸들러가 커버하지만 graceful 하지 않음, 성능 저하, 잠재적 버그.
- **Info** — 개선 권장, 코드 스타일, 관례 불일치.
</severity>

심각도 판정 시 시스템적 맥락 확인:

- 상위 보호장치 존재 여부 (Flask `@app.errorhandler(Exception)`, Celery `SqlAlchemyTask` 기본 세션 관리, 글로벌 미들웨어 등) — 커버 시 최대 Warning. "커버는 되지만 graceful 하지 않다" vs "전혀 커버 안 됨" 구분.
- 양호 패턴 섹션과 발견 항목 심각도가 *논리적으로 일관*되는지 — 모순이면 해소.
- 동일 패턴의 기존 해결책 Grep — 있으면 수정 난이도 하향 + 명시.
- 해당 파일·함수가 실제로 import 되어 런타임에 실행되는지 확인 — *dead code 에는 Critical 부여 안 함*.
- 코드에 없는 내용은 추측 추가 안 함 (예: "deprecated 예정" 등 근거 없는 문구).

---

## Phase 2: 검증 루프 (최대 3회)

초안 생성 후 검증 서브에이전트를 투입한다.

### 검증 모드 — 항상 vs 해당 시

매 iteration 마다 **항상 검증**할 항목 (1~3) 과, 발견 항목이 해당 분류에 속할 때 *해당 시* 검증할 항목 (4~12) 으로 분리:

<verification_steps>
**항상 검증 (모든 iteration)**:

1. **경로 검증** — 발견 항목의 `파일:라인` 이 실제 존재하는지 Glob/Read 확인
2. **코드 검증** — 코드 스니펫이 현재 코드와 일치하는지 Read 로 대조
3. **요약-상세 일관성** — 요약 테이블과 상세 헤더의 심각도 일치

**해당 시 검증**:

4. **맥락 검증** — 주변 코드 read, 진짜 문제인지 판단 (예: N+1 처럼 보이지만 prefetch 적용됨)
5. **시스템 맥락** — 상위 보호장치가 이슈를 커버하면 심각도 하향
6. **내부 모순** — 양호 패턴과 발견 항목의 심각도 논리 일관성
7. **인접 / 형제 패턴 스캔** — 발견 지점 ±20줄 + 같은 파일 다른 함수 + 같은 디렉토리 형제 파일에서 동일 패턴
8. **누락 탐지** — 패턴 변형이나 다른 디렉토리에서 놓친 항목 추가 스캔
9. **제안 검증** — 수정 제안이 기존 유틸 활용·함수 모든 분기 커버
10. **이전 피드백 반영** — 이전 분석 피드백 지적 사항이 반영되었는지
11. **미사용 판정** — "미사용" 주장 시 프로젝트 전체 (소스+테스트+스크립트+설정) Grep 필수. 같은 파일 내 사용 포함. Grep 결과가 정의 외 0건일 때만 "미사용" 인정
12. **미존재 판정** — "미존재 / 없음 / 삭제됨" 주장 시 Glob 으로 실제 파일 존재 확인. *import 주석 처리 ≠ 파일 미존재*. Glob 0건일 때만 "미존재" 인정
</verification_steps>

### 변경 리스트 형식

검증 에이전트는 다음 JSON 형식으로 변경 리스트를 반환한다 (자유 형식 텍스트는 후속 iteration 일관성 ↓):

```json
{
  "iteration": 1,
  "changes": [
    {"action": "remove", "id": "#3", "reason": "상위 errorhandler 가 커버 — Critical 자격 없음"},
    {"action": "modify", "id": "#5", "field": "심각도", "from": "Critical", "to": "Warning", "reason": "..."},
    {"action": "add", "summary": "...", "reason": "인접 형제 파일에서 동일 패턴 발견"}
  ],
  "rounds_remaining": 2
}
```

### 루프 동작

```
검증 에이전트 → JSON 변경 리스트
  │
  ├── changes 비어있음 → 즉시 확정
  └── changes 있음 → 문서에 반영 → 다시 검증 (최대 3회)
```

종료 조건: **changes 비어있음** OR **3회 도달** (먼저 도달 쪽).

---

## 출력

### 파일 위치

`docs/analysis/deep/{topic-slug}.md`

### 슬러그 결정

사용자가 인자에 영문 슬러그 명시 (`/jean:deep-analyze "에러 핸들링 누락" --slug error-handling-gaps`) → 그대로 사용.

명시 없을 시:
1. Claude 가 영문 슬러그 제안 (예: "에러 핸들링 누락" → `error-handling-gaps`)
2. 사용자 한 번 확인 (`AskUserQuestion`) 후 확정

룰: 한글 의미 → 영문 직역, 공백 → 하이픈, 소문자.

### 출력 템플릿

<output_format>
```markdown
# {주제} 심층 분석

> 생성일: {YYYY-MM-DD} | 프로젝트: {프로젝트명}
> 스택: {감지된 스택}
> 분석 질문: "{사용자의 원래 질문}"
> 스캔 범위: {스캔한 디렉토리}

## TL;DR

- 발견 항목: {N}개 (Critical {N}, Warning {N}, Info {N})
- 핵심 발견: {가장 중요한 1-2개 요약}

## 발견 항목 요약

| # | 심각도 | 파일:라인 | 설명 | 수정 난이도 |
|---|--------|----------|------|------------|

## 상세 분석

### #{N}. [{심각도}] {제목}

**위치**: `{파일경로}:{라인}`

**문제 코드**:
\`\`\`{lang}
{코드 스니펫 ±5줄}
\`\`\`

**왜 문제인가**: {설명}

**영향 범위**: {해당 시}

**수정 제안**:
\`\`\`{lang}
{수정 코드 예시}
\`\`\`

**수정 난이도**: {Easy / Medium / Hard}

## 검증 과정에서 확인된 양호 패턴 (선택)

{false positive 로 제거된 항목의 이유 기록}

---

> 검증: {N}회 수행 | 제거 {N}건, 수정 {N}건, 추가 {N}건
```
</output_format>

### `_INDEX.md` 와의 관계

`/jean:deep-analyze` 의 산출물은 `docs/analysis/deep/` 서브디렉토리에 둠으로써 `/jean:analyze` 의 6개 리포트 (`01~06.md` + `_INDEX.md`) 와 분리된다. `_INDEX.md` 는 deep 결과를 자동 포함하지 않는다 — 두 스킬은 독립적으로 운영된다.

deep 결과를 `_INDEX.md` 에서 참조하려면 사용자가 수동으로 추가하거나, 별도로 `docs/analysis/deep/_INDEX.md` 를 만들어 deep 결과만 모은다.

### 콘솔 요약

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  심층 분석 완료: {주제}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  발견: {N}개 (Critical {N}, Warning {N}, Info {N})
  검증: {N}회 (제거 {N}, 수정 {N}, 추가 {N})

  결과: docs/analysis/deep/{slug}.md
  백업: docs/analysis/deep/_backup_{timestamp}/ (재실행 시 자동 보존)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 출력 예시 (worked example)

가상 fixture (FastAPI bookmarks API) 에서 "에러 핸들링 누락" 주제로 분석한 결과의 일부 — `~/.claude/skills/jean-deep-analyze/references/example-error-handling.md` 참고.

<example>
샘플 발견 항목 한 개 (간략):

```markdown
### #1. [Warning] BookmarkService.delete — 외부 commit 실패 시 미캐치

**위치**: `app/services/bookmark_service.py:28-30`

**문제 코드**:
[28-30줄 스니펫]

**왜 문제인가**: `await self.session.commit()` 이 DB connection 끊김·deadlock 등으로 실패할 수 있으나 try/except 없음. 글로벌 errorhandler 가 500 으로 커버하긴 하지만 graceful 하지 않음 (사용자에게 의미 있는 메시지 미제공).

**수정 제안**: try/except 로 commit 감싸고 `IntegrityError`·`OperationalError` 분기 처리.

**수정 난이도**: Easy
```

→ 톤·디테일·심각도 판정의 reference. 분량 / 항목 수는 fixture 크기에 따라 조절.
</example>

---

## 안내

- 발견 항목은 코드로 확인한 것만 기재 (추측·가정 표현 대신 `[확인 필요]` 마커).
- 파일 경로는 실제 존재하는 경로만 인용 (Phase 2 검증으로 보장).
- 환경변수 값은 변수명만 기재.
- 검증 에이전트의 변경 리스트는 JSON 형식 — 자유 텍스트 시 후속 iteration 일관성 깨짐.
