# claude-custom-skills

Claude Code 개인 커스텀 스킬 모음 (`jean:*` 네임스페이스).

스킬을 처음부터 만들고 다듬고 평가하는 **메타 스킬**과, 코드베이스를 분석·정리·자동화하는 **유틸리티 스킬**로 구성된다.

파일은 `.claude/commands/jean:*.md` (슬래시 커맨드)와 `.claude/skills/jean-*/` (보조 자산)에 위치한다. 커맨드 파일명에 콜론(`:`)이 포함되어 있어 Windows 네이티브 파일시스템에서는 체크아웃되지 않는다 — WSL 환경 사용을 권장.

## 스킬 카탈로그

### 오케스트레이터

#### `/jean:skill`

무엇이든 만들 때 가장 먼저 호출하는 통합 오케스트레이터. 요구사항을 분석해 `skill-creator` / `rule-creator` / `hook-creator` / `claude-md-creator` / `agent-creator` 중 필요한 도구를 자동 선택·순서 결정해 실행한다. 새 기능을 처음부터 만들거나, 기존 스킬을 올바른 프리미티브(Skill / Rule / Hook / CLAUDE.md / Agent)로 리팩터링할 때 사용. 스킬 생성 시 eval pass_rate ≥ 0.95 도달까지 자동 루프, 모든 수정 시 사이드 이펙트 자동 검사.

- 인자: `{만들고 싶은 기능/행동 설명 or 리팩터링할 스킬 경로}`
- 트리거: "만들어줘", "추가해줘", "이 스킬 정리해줘", "자동화해줘", "어떤 프리미티브에 넣어야 해"

### 프리미티브 생성기

`jean:skill` 이 위임하는 도구들. 단독 호출도 가능하다.

#### `/jean:skill-creator`

새 Claude 스킬을 만들거나 기존 스킬을 개선·평가한다. Anthropic 공식 prompting best practices (XML 태그, 긍정 프레이밍, few-shot 3원칙, 골든룰)를 가이드에 통합. eval 루프로 description triggering 정확도까지 측정. 스킬을 처음부터 만들거나 기존 스킬의 description/구조를 다듬거나 성능을 측정할 때 사용.

- 인자: `{스킬 이름 or 설명, 비워두면 대화로 진행}`

#### `/jean:agent-creator`

Claude Code 에이전트(`.claude/agents/<name>.md`)를 설계·작성·검증한다. Skill 과 달리 에이전트는 오케스트레이터가 위임하는 격리 전문가 — 모델 선택, 도구 권한 최소화, 입출력 계약 설계에 집중. eval 루프로 품질 검증.

- 인자: `{에이전트 역할/이름 설명, 비워두면 대화로 진행}`
- 트리거: "서브에이전트 설계", "전문가 위임 구조", "병렬 처리용 에이전트", "격리 컨텍스트 필요"

#### `/jean:rule-creator`

기존 스킬을 분석해 rule (`CLAUDE.md` 또는 `.claude/rules/*.md`) 로 옮기면 좋은 부분을 찾아 제안·생성·정리. 기존 rule 과 중복/모순 자동 검출. 스킬에서 **추출**하는 도구이므로 분석 대상 스킬이 먼저 존재해야 한다 (rule 을 처음부터 만들 때는 파일을 직접 편집).

- 인자: `{스캔 대상 경로 or 비워서 기본 스캔}`

#### `/jean:hook-creator`

기존 스킬을 분석해 hook (`settings.json` 또는 `SKILL.md` frontmatter) 로 옮기면 좋은 결정적 동작을 찾아 제안·생성·검증·정리. 기존 hook 과 중복/모순 자동 검출, dry-run 검증과 단계적 설치(local-first)로 안전성 확보. hook 을 처음부터 만들 때는 `.claude/settings.json` 을 직접 편집.

- 인자: `{스캔 대상 경로 or 비워서 기본 스캔}`

#### `/jean:claude-md-creator`

프로젝트를 분석해 Claude Code 최적화 `CLAUDE.md` 를 생성하거나 기존 파일을 개선. 팀 컨벤션 정의, 신규 프로젝트 셋업, 기존 파일 정리 모두 커버. 새 프로젝트 시작 시 가장 먼저 사용할 것.

- 인자: `{프로젝트 경로 or 비워서 현재 디렉토리}`
- 트리거: "CLAUDE.md 만들어줘", "프로젝트 컨텍스트 주입", "Claude 가 프로젝트를 알게 해줘"

### 정합성 도구

#### `/jean:skill-sync`

skills/agents/rules 파일을 수정·추가한 뒤 일관성 자동 점검·정리 스킬. 포맷 준수, 스킬 간 상호작용 계약, rules paths 불일치, 잔제 패턴, 유령 참조, 중복 내용을 탐지·수정. 단독 실행하거나 `jean:skill` 완료 후 자동 호출됨. 인자로 파일명을 넘기면 해당 파일 관련 범위만 스캔.

- 인자: `{파일명 or 비워서 전체 스캔}`

### 코드베이스 분석

#### `/jean:analyze`

프로젝트 코드베이스 전체를 자동 스캔해 6개 영역 리포트(아키텍처·기능·워크플로우·데이터·건강도·온보딩)를 `docs/analysis/` 에 생성. 스택 자동 감지(Python · Node · Go · Rust · Java/Kotlin · Ruby) + 프레임워크별 패턴 추출. 신입 온보딩 문서, 아키텍처 감사, 코드 건강도 점검에 적합.

- 인자: `[all|architecture|features|workflow|data|health|onboarding]` (기본: `all`)
- 트리거: "이 프로젝트 어떻게 돌아가", "아키텍처 감사", "온보딩 가이드 만들어줘"

#### `/jean:deep-analyze`

주제 자유형 심층 분석. 사용자가 지정한 이슈(N+1 쿼리, 중복 코드, 에러 핸들링, dead code, 보안 취약점, 성능 핫스팟 등 무엇이든)에 대해 코드 전수 조사 후 파일:라인 단위 상세 리포트 생성. 검증 루프(최대 3회)로 false positive 제거.

- 인자: `{분석 주제 — 자유 형식}`
- 트리거: "어디서 발생하는지", "찾아줘", "누락된 곳", "감사", "이슈 추적"

### 사고 정리

#### `/jean:idea`

두서없는 생각·아이디어를 대화로 함께 다듬어 LLM 이 효율적으로 실행할 수 있는 구조화된 명령으로 변환. 주제 무관 — 기술 구현, 설계 고민, 리서치, 비즈니스 기획, 문제 해결 모두 지원. "뭔가 좀 모호한데", "어디서부터 시작할지" 같은 막연한 입력에 사용.

- 인자: `{당신의 생각 — 자유 형식}`
- 트리거: "구체화", "정리해줘", "brainstorm", "막연"

#### `/jean:tick-tock`

사용자와 나눈 느슨한 수정 논의(대화, 코멘트, 생각 덩어리)를 어떤 AI 실행자에도 바로 붙여넣을 수 있는 명확한 수정 브리핑으로 압축. 수정 지시가 모호하거나 흩어져 있을 때, AI 에게 정확히 뭘 어떻게 고치라고 전달할지 막막할 때 사용. 독립 스킬 — jean 생태계 오케스트레이터와는 연동하지 않음.

- 인자: `{수정 논의 내용 또는 수정 지시 — 자유 텍스트}`

## 지원 스킬 폴더

세 스킬은 단일 `.md` 파일 외에 `.claude/skills/` 하위 보조 자산을 사용한다.

| 폴더 | 용도 |
|---|---|
| `jean-analyze/references/` | 6개 영역 리포트 템플릿, 예시(`example-fastapi-mini/`) |
| `jean-deep-analyze/references/` | 검증 루프 예시 (error-handling 등) |
| `jean-skill-creator/` | eval 자동화 스크립트(`scripts/`), 평가 에이전트(`agents/`), eval viewer, 평가 기준(`references/schemas.md`), LICENSE |

## 사용 권장 순서

1. 새 프로젝트 시작 → `/jean:claude-md-creator` 로 컨텍스트 셋업
2. 막연한 아이디어 → `/jean:idea` 로 명확화
3. 무엇이든 만들기 → `/jean:skill` (자동으로 적절한 creator 위임)
4. 기존 코드베이스 파악 → `/jean:analyze` (개요) → `/jean:deep-analyze` (이슈 추적)
5. AI 에게 수정 지시 정리 → `/jean:tick-tock`
6. 스킬 수정 후 → `/jean:skill-sync` 자동 정합성 점검

## 라이선스

`jean:skill-creator` 는 Anthropic Skill Creator 기반 작업물을 포함하며 해당 LICENSE 는 `.claude/skills/jean-skill-creator/LICENSE.txt` 에 포함된다. 그 외 자체 작성 스킬은 별도 명시 전까지 개인 사용 목적.
