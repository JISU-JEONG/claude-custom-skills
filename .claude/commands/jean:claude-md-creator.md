---
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
argument-hint: "{프로젝트 경로 or 비워서 현재 디렉토리}"
description: 프로젝트를 분석해 Claude Code 최적화 CLAUDE.md를 생성하거나 기존 파일을 개선한다. "CLAUDE.md 만들어줘", "클로드에게 프로젝트 컨텍스트 주입해줘", "컨텍스트 파일 만들어줘", "새 프로젝트 셋업해줘", "프로젝트 설명 파일 만들어줘", "Claude가 프로젝트를 알게 해줘" 등 모든 CLAUDE.md 생성·갱신 요청에 사용. 팀 컨벤션 정의, 신규 프로젝트 셋업, 기존 파일 개선 모두 커버. 새 프로젝트 시작 시 가장 먼저 사용할 것.
---

# jean:claude-md-creator

프로젝트를 분석해 Claude Code에 최적화된 CLAUDE.md를 생성하는 도구.

기본 응답 언어는 한국어. 사용자가 영어로 답하면 따라간다.

---

## 동작 흐름

5단계: **Discover → Analyze → Draft → Review → Write**

각 단계는 다음으로 진행하기 전에 사용자에게 결과를 보여준다. Draft 이후 피드백이 있으면 반영 후 즉시 Write로 넘어간다.

---

## Step 1. Discover

<instructions>
**경로 결정:**
- 인자로 경로가 왔으면 그 경로를 대상으로 한다
- 없으면 현재 작업 디렉토리 기준

**기존 CLAUDE.md 확인 (업데이트 모드 vs 신규 생성 모드 결정):**
```bash
ls -la CLAUDE.md .claude/CLAUDE.md CLAUDE.local.md 2>/dev/null
```
파일이 존재하면 **업데이트 모드** — 기존 파일을 먼저 Read해 이미 있는 내용을 파악한다.
파일이 없으면 **신규 생성 모드**.

**프로젝트 구조 스캔:**
```bash
ls -la                                                # 루트 파일
find . -maxdepth 2 -type d -not -path '*/node_modules/*' -not -path '*/.git/*' | sort
```

수집 대상 파일 (존재하는 것만):
- 패키지 매니저: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`
- 문서: `README.md`, `README.ko.md`
- 린터·포매터: `.eslintrc*`, `.prettierrc*`, `ruff.toml`, `.flake8`, `biome.json`, `.editorconfig`
- 빌드·실행: `Makefile`, `justfile`, `docker-compose.yml`, `Dockerfile`
- 환경: `.env.example`, `.env.sample`
- TypeScript: `tsconfig.json`
- 테스트: `jest.config.*`, `vitest.config.*`, `pytest.ini`, `conftest.py`

사용자에게 스캔 결과 한 줄 요약: "프로젝트 스캔 완료 — {감지된 주요 파일 목록}. 분석 들어갑니다."
</instructions>

---

## Step 2. Analyze

<instructions>
Step 1에서 찾은 파일을 읽고 아래 항목을 추출한다. 존재하는 파일만 읽는다.

### 추출 항목

**기술 스택 & 언어**
- 메인 언어·프레임워크·런타임 버전
- 예: "TypeScript 5.x + Next.js 14, Node 20", "Python 3.12 + FastAPI + SQLAlchemy"

**핵심 명령어**
- `package.json` scripts, Makefile targets, `pyproject.toml` scripts에서 추출
- 개발 서버, 빌드, 테스트, 린트 명령 + 포트 번호 포함

**디렉토리 아키텍처**
- 핵심 디렉토리 역할 (src/, app/, api/, tests/, packages/ 등)
- 모노레포면 패키지 구조 파악

**코드 스타일 & 컨벤션**
- 들여쓰기 방식 (2 space, 4 space, tab)
- 명명 규칙 (camelCase, snake_case, PascalCase)
- 포매터·린터 핵심 설정 요약

**주의사항 후보**
- `.env.example` → 필수 환경변수 목록
- README의 IMPORTANT / NOTE / 주의 / 경고 항목
- docker-compose의 필수 서비스 의존성

**@import 후보**
- `docs/`, `docs/api/`, `architecture.md` 같은 별도 문서가 있으면 @import 활용 권장
- 300줄 제한을 넘길 것 같으면 무거운 내용은 @import로 분리
</instructions>

---

## Step 3. Draft

<instructions>
분석 결과를 바탕으로 CLAUDE.md 초안을 작성한다.

### 구조 템플릿

```markdown
# {프로젝트명}

{한 줄 프로젝트 설명}

## 코드 스타일
- {구체적 스타일 규칙}

## 명령어
- `{cmd}`: {설명 + 포트 번호}

## 아키텍처
- `{dir}/`: {역할}

## 주의사항
- IMPORTANT: {중요 제약 또는 민감 파일}
- {환경변수·서비스 의존성}
```

### 작성 원칙

- **300줄 이하** — CLAUDE.md는 매 세션 자동으로 로드되므로 토큰 비용을 고려한다
- **사실만** — 코드베이스에서 확인된 내용만 작성한다. 불확실한 내용은 "[확인 필요]" 표시 후 사용자에게 확인
- **실행 가능한 정보** — "코드를 깔끔하게 써라" 대신 "2-space 들여쓰기, named export만 사용" 같은 구체적 규칙
- **@import 활용** — 별도 문서가 있으면 직접 복사하지 않고 `@docs/xxx.md` 참조

**신규 생성**: 초안 전체를 보여준다.
**업데이트 모드**: 추가·변경되는 부분만 diff 형태로 보여준다. 기존 내용을 함부로 삭제하지 않는다.
</instructions>

---

## Step 4. Review

<instructions>
초안을 사용자에게 제시하고 피드백을 받는다.

체크 질문:
- 빠진 팀 규칙이나 프로젝트 특수 제약이 있나요?
- 추가할 명령어나 주의사항이 있나요?
- 저장 위치는 `{경로}/CLAUDE.md`로 맞나요?

**스코프 선택지 제시 (사용자가 묻지 않아도 한 번 확인):**

| 위치 | 용도 |
|---|---|
| `CLAUDE.md` (프로젝트 루트) | 팀 공유 — git 커밋 대상 (기본 권장) |
| `.claude/CLAUDE.md` | 프로젝트 전체, `.claude/` 구조 선호 시 |
| `CLAUDE.local.md` | 본인 전용 — gitignore 대상 |
| `~/.claude/CLAUDE.md` | 모든 프로젝트에 전역 적용 |

기본값: **프로젝트 루트 `CLAUDE.md`**. 사용자 동의 확인 후 Step 5.
</instructions>

---

## Step 5. Write

<instructions>
사용자 승인 후 파일을 작성한다.

**신규 생성:**
Write 도구로 결정된 경로에 작성.

**업데이트 모드:**
1. 기존 파일 Read
2. 변경 내용 머지 (기존 섹션 보존, 새 섹션 추가, 충돌 시 사용자 판단)
3. diff를 사용자에게 한 번 더 확인
4. Edit 또는 Write 적용

**완료 보고:**
```
✅ {경로} 작성 완료
- 줄 수: {N}줄
- 포함 섹션: {목록}
```

줄 수가 300줄을 넘으면: "300줄 초과 — 무거운 섹션을 @import로 분리하거나 `.claude/rules/`로 추출하는 것을 권장합니다."

**후속 제안:**
- 여러 스킬에 반복되는 팀 컨벤션이 있으면 → `/jean:rule-creator` 실행 권장
- 파일 저장·커밋 등 이벤트 기반 자동화가 필요하면 → `/jean:hook-creator` 실행 권장
</instructions>
