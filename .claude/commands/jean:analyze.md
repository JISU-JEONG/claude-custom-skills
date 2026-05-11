---
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
argument-hint: "[all|architecture|features|workflow|data|health|onboarding]"
description: 프로젝트 코드베이스 전체를 자동 스캔해 6개 영역(아키텍처·기능·워크플로우·데이터·건강도·온보딩) 리포트를 `docs/analysis/` 에 생성한다. 스택 자동 감지 (Python·Node·Go·Rust·Java/Kotlin·Ruby) + 프레임워크별 패턴 추출. 코드베이스 파악, 신입 온보딩 문서, 아키텍처 감사, 코드 건강도 점검, "이 프로젝트 어떻게 돌아가" 류 질문에 호출. 키워드 — 분석 리포트, 프로젝트 개요, 아키텍처 감사, 온보딩 가이드, 코드베이스 파악, project overview, architecture audit, code health, onboarding doc.
---

# /jean:analyze

현재 작업 디렉토리의 프로젝트 전체를 분석해 `docs/analysis/` 에 구조화된 리포트를 생성한다. 타 도메인 개발자의 코드 파악·신입 온보딩·버그 추적 용도.

스택과 구조를 자동 감지하므로 어떤 프로젝트에서든 동작한다.

## 인자

단일 인자만 받는다.

| 인자 | 동작 |
|------|------|
| (없음) 또는 `all` | 전체 분석 (해당하는 섹션 + `_INDEX.md`) |
| `architecture` | `01-architecture.md` 만 |
| `features` | `02-feature-map.md` 만 |
| `workflow` | `03-workflow.md` 만 |
| `data` | `04-data-model.md` 만 |
| `health` | `05-health.md` 만 |
| `onboarding` | `06-onboarding.md` 만 |
| 그 외 | 사용 가능 인자 목록을 출력하고 종료 |

## 핵심 원칙

<principles>
1. **코드 기반 진실** — 모든 내용은 실제 코드를 Glob/Grep/Read 로 직접 추출한다. 추측한 내용에는 `[확인 필요]` 마커를 단다 (false positive 방지).
2. **TL;DR 우선** — 모든 리포트 상단에 핵심 요약을 배치한다.
3. **추적 가능 경로** — 파일 경로는 프로젝트 루트 기준 상대 경로로 기재한다.
4. **백업 후 덮어쓰기** — 기존 `docs/analysis/` 가 있으면 `docs/analysis/_backup_YYYYMMDD-HHMMSS/` 로 옮긴 뒤 재생성한다 (사용자 작업 손실 방지).
5. **에러 허용** — Read 실패 시 `[분석 불가]`, Bash 실패 시 `N/A` 로 표시하고 진행한다.
6. **독립 섹션** — 각 리포트는 단독으로 읽힐 수 있어야 한다. 다른 리포트 인용 시 상대 링크를 쓴다.
7. **해당 없으면 스킵** — 프로젝트에 해당하지 않는 섹션은 생성하지 않는다 (예: DB 없으면 04 스킵).
8. **민감 정보 보호** — `.env` 값은 변수명만 기재하고 값은 마스킹한다.
9. **용어 보존** — 도메인 특화 용어는 코드에 쓰인 그대로 유지한다 (의역 시 추적 불가).
10. **생성물 디렉토리 제외** — `node_modules/`, `__pycache__/`, `.git/`, `vendor/`, `dist/`, `build/`, `target/`, `.gradle/`, `.venv/`, `.next/`, `out/` 는 분석 대상에서 빼고 시작한다.
</principles>

## 출력 디렉토리

`docs/analysis/`. 디렉토리 부재 시 생성. 기존 파일은 `docs/analysis/_backup_{timestamp}/` 로 이동.

---

## Phase 0: 프로젝트 Discovery

전체 분석 시 항상 먼저 실행. 개별 섹션 호출 시도 해당 섹션이 의존하는 부분만 수행한다.

### 0.1 언어 / 패키지 매니저 감지

lock 파일과 manifest 로 스택 + 패키지 매니저를 한 번에 판별:

<detection>
| 감지 파일 | 언어 | 패키지 매니저 |
|-----------|------|---------------|
| `pyproject.toml` + `poetry.lock` | Python | poetry |
| `pyproject.toml` + `uv.lock` | Python | uv |
| `pyproject.toml` (lock 없음) | Python | pip / pdm |
| `requirements.txt` | Python | pip |
| `setup.py` | Python | setuptools |
| `package.json` + `pnpm-lock.yaml` | Node.js / TypeScript | pnpm |
| `package.json` + `yarn.lock` | Node.js / TypeScript | yarn |
| `package.json` + `package-lock.json` | Node.js / TypeScript | npm |
| `package.json` + `bun.lockb` | Node.js / TypeScript | bun |
| `go.mod` | Go | go mod |
| `Cargo.toml` | Rust | cargo |
| `pom.xml` | Java | maven |
| `build.gradle` 또는 `build.gradle.kts` | Java / Kotlin | gradle |
| `Gemfile` | Ruby | bundler |
</detection>

### 0.2 프레임워크 / ORM 감지

언어별 import 또는 dependency 검색.

**Python** (Grep import 패턴):
- `from fastapi` → FastAPI
- `from flask` → Flask
- `from django` → Django
- `from langgraph` → LangGraph
- `from langchain` → LangChain
- `from sqlalchemy` → SQLAlchemy
- `from tortoise` → Tortoise ORM

**Java / Kotlin** (`build.gradle*` 또는 `pom.xml` Read → dependencies):
- `spring-boot` → Spring Boot
- `spring-webflux` → Spring WebFlux
- `spring-data-jpa` → Spring Data JPA
- `exposed` → Exposed (Kotlin ORM)
- `mybatis` → MyBatis

**Node.js** (`package.json` Read → dependencies):
- `next` → Next.js
- `express` → Express
- `nestjs` → NestJS
- `prisma` → Prisma ORM
- `drizzle-orm` → Drizzle ORM

**Go** (`go.mod` Read → require):
- `gin-gonic/gin` → Gin
- `labstack/echo` → Echo
- `gorm.io/gorm` → GORM

**Rust** (`Cargo.toml` Read → dependencies):
- `actix-web` → Actix Web
- `axum` → Axum
- `diesel` → Diesel ORM
- `sea-orm` → SeaORM

### 0.3 프로젝트 구조 감지

Glob / Bash 로 디렉토리 구조 파악:

```bash
# 소스 루트 후보
ls src/ app/ lib/ cmd/ internal/ pkg/ main/ 2>/dev/null
# JVM
ls src/main/java/ src/main/kotlin/ 2>/dev/null
```

| 감지 항목 | 확인 방법 | 영향 |
|-----------|----------|------|
| 소스 루트 | `src/`, `app/`, `lib/`, `cmd/`, `main/`, `src/main/java/`, `src/main/kotlin/` 등 존재 여부 | 모든 섹션의 탐색 경로 |
| API 레이어 | `controllers/`, `routes/`, `handlers/`, `api/`, `views/`, `presentation/` 존재 | 섹션 02 활성화 |
| 서비스 레이어 | `services/`, `usecases/`, `application/`, `domain/` 존재 | 섹션 01, 02 |
| 데이터 레이어 | `repositories/`, `models/`, `entities/`, `schemas/`, `prisma/`, `domain/` 존재 | 섹션 04 활성화 |
| 워크플로우 / 파이프라인 | `graph/`, `workflows/`, `pipelines/`, `agents/`, `tasks/`, `jobs/` 존재 | 섹션 03 활성화 |
| 테스트 | `tests/`, `test/`, `__tests__/`, `spec/`, `src/test/` 존재 | 섹션 05 |
| 설정 | `settings.py`, `.env*`, `config/`, `application.yml`, `application.properties` 존재 | 섹션 01 |
| DI 컨테이너 | `containers.py`, `di/`, `injection/`, `providers/`, Spring `@Configuration` 클래스 | 섹션 01 |
| 예외 / 에러 | `exceptions/`, `errors/` 존재 | 섹션 01 |
| `CLAUDE.md` | 프로젝트 루트 또는 `.claude/` 하위 | 섹션 06 |

### 0.4 Discovery 결과 메모

감지 결과를 다음 형태로 *작업 컨텍스트에 들고 간다*. 별도 파일 저장 없이 각 섹션 시작에서 짧게 다시 인용한다.

```yaml
detected_stack:
  language: {Python|Node.js|Go|...}
  framework: {FastAPI|Next.js|...}
  orm: {SQLAlchemy|Prisma|...}
  package_manager: {poetry|pnpm|npm|...}
  source_root: {src/|app/|...}
  has_api: true|false
  has_services: true|false
  has_data_layer: true|false
  has_workflow: true|false
  has_tests: true|false
  has_di: true|false
```

---

## 섹션별 분석 지침

각 섹션의 분석 순서·출력 포맷은 다음 reference 파일을 Read 해서 따른다:

| 섹션 | Reference | 생성 조건 |
|------|-----------|----------|
| 01 아키텍처 | `~/.claude/skills/jean-analyze/references/section-01-architecture.md` | 항상 |
| 02 기능 매핑 | `~/.claude/skills/jean-analyze/references/section-02-feature-map.md` | `has_api = true` |
| 03 워크플로우 | `~/.claude/skills/jean-analyze/references/section-03-workflow.md` | `has_workflow = true` |
| 04 데이터 모델 | `~/.claude/skills/jean-analyze/references/section-04-data-model.md` | `has_data_layer = true` |
| 05 건강도 | `~/.claude/skills/jean-analyze/references/section-05-health.md` | 항상 |
| 06 온보딩 | `~/.claude/skills/jean-analyze/references/section-06-onboarding.md` | 항상 |

각 reference 는 독립적으로 작성되어 있어 해당 섹션만 호출될 때도 reference 한 개만 Read 하면 된다 (progressive disclosure).

---

## `_INDEX.md` 갱신

모든 섹션 생성 완료 후 `docs/analysis/_INDEX.md` 갱신:

```markdown
# 프로젝트 분석 리포트

> **프로젝트**: {프로젝트명}
> **스택**: {감지된 스택 요약}
> **생성일**: {YYYY-MM-DD HH:MM}
> **분석 범위**: {전체 또는 개별 섹션명}

## 요약 통계

| 지표 | 값 |
|------|-----|
| 소스 파일 | {N}개 |
| 테스트 파일 | {N}개 |
| API 엔드포인트 | {N}개 또는 N/A |
| 워크플로우 노드 | {N}개 또는 N/A |
| DB 모델 | {N}개 또는 N/A |
| TODO/FIXME | {N}개 |

## 리포트 목차

| # | 리포트 | 설명 | 생성 여부 |
|---|--------|------|----------|
| 01 | [아키텍처](./01-architecture.md) | 레이어, DI, 설정 | 항상 |
| 02 | [기능 매핑](./02-feature-map.md) | API 추적 맵 | API 있을 때 |
| 03 | [워크플로우](./03-workflow.md) | 파이프라인/그래프 | 워크플로우 있을 때 |
| 04 | [데이터 모델](./04-data-model.md) | 엔티티/관계 | DB 있을 때 |
| 05 | [건강도](./05-health.md) | 통계, TODO, 테스트 | 항상 |
| 06 | [온보딩](./06-onboarding.md) | 환경세팅, 작업가이드 | 항상 |

## 재생성 방법

| 명령어 | 설명 |
|--------|------|
| `/jean:analyze` | 전체 재생성 |
| `/jean:analyze architecture` | 아키텍처만 |
| `/jean:analyze features` | 기능 매핑만 |
| `/jean:analyze workflow` | 워크플로우만 |
| `/jean:analyze data` | 데이터 모델만 |
| `/jean:analyze health` | 건강도만 |
| `/jean:analyze onboarding` | 온보딩 가이드만 |
```

---

## 콘솔 요약 출력

분석 완료 후 사용자에게 다음 형태로 요약을 보여준다:

<output_format>
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  프로젝트 분석 완료
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  스택: {language} + {framework}

  생성된 리포트:
  ✅ docs/analysis/{생성된 파일들}
  ⏭️ {스킵된 섹션} (해당 없음)

  요약: 소스 {N}개 | API {N}개 | TODO {N}개

  백업: docs/analysis/_backup_{timestamp}/ (재실행 시 자동 보존)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
</output_format>

---

## 출력 예시

`/jean:analyze` 가 어떤 결과를 만들어야 하는지 보여주는 worked example 한 세트:

```
~/.claude/skills/jean-analyze/references/example-fastapi-mini/
├── README.md           # 가상 fixture (FastAPI todo API) 의 구조·코드
├── 01-architecture.md  # 아키텍처 분석 결과
├── 02-feature-map.md   # 기능 매핑 결과
└── 06-onboarding.md    # 온보딩 가이드 결과
```

<example>
가상 fixture 가 5파일짜리 단순 FastAPI todo API (FastAPI + SQLAlchemy + PostgreSQL) 일 때, 위 3개 분석 결과의 톤·디테일·구조가 기준이 된다. 실제 분석 시에는 사용자 프로젝트의 진짜 코드를 Glob/Grep/Read 로 수집해 같은 형태로 채운다.

핵심 패턴:
- TL;DR 은 5줄 이내, 스택·레이어·DI·인증·DB 5개 키 항목으로 정리
- 표는 *실제 파일 경로:함수* 까지 추적 가능하게 작성
- 추적 맵은 `핸들러 → 서비스 → 데이터 → 응답` 4단계 다이어그램 + 관련 파일 표
- 온보딩의 "이것부터 읽어라" 표는 7개 이내 핵심 파일로 한정
</example>

분량·톤 조절 시 이 example 을 비교 reference 로 사용한다.

---

## 안내

- 환경변수 값은 변수명만 기재한다 (보안).
- 도메인 특화 용어는 코드에 쓰인 그대로 유지한다.
- 확인 불가 항목은 `[확인 필요]` 마커를 단다 (추측 표현 대신).
- 파일 경로는 실제 존재하는 경로만 인용한다.
- 각 리포트는 A4 5~15페이지 분량을 목표로 한다.
