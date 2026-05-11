# claude-custom-skills

Claude Code 개인 커스텀 스킬·커맨드 모음.

파일 구조:

- `.claude/commands/` — 슬래시 커맨드 (`/jean:*`)
- `.claude/skills/` — 스킬 본체 또는 커맨드의 보조 자산

커맨드 파일명에 콜론(`:`)이 포함되어 있어 Windows 네이티브 파일시스템에서는 체크아웃되지 않는다. WSL 환경 사용을 권장.

## 스킬 목록

### cal

일정·약속·미팅 관리. 추가·조회·삭제·고정 명령어 지원, 자연어 날짜(내일/모레/오늘) 처리. 데이터는 `~/.jean/schedule/schedules.json` 에 저장되어 외부 status line 등에서 참조 가능. 일정을 잡거나 다가오는 미팅을 확인할 때 사용.

- 인자: `add 내일 14:00 팀미팅` / `list [--all]` / `rm <id>` / `pin <id>` / `unpin <id>` / `past`
- 트리거: 일정, 약속, 미팅, 캘린더, schedule, appointment, remind

### dev-spec

요구사항을 협업 가능한 작업 단위(Work Unit)로 분해하고 각 WU 별 상세 개발 기획서를 자동 생성. Notion 페이지, MD 파일, 직접 텍스트 입력 모두 지원. 한 번에 전부 생성하지 않고 WU 하나씩 사용자 피드백을 받으며 진행한다.

- 인자: `notion <page>` / `file <path>` / `<직접 입력 텍스트>`
- 트리거: dev-spec, 개발 기획서, 설계문서 작성, 태스크 분해, work unit breakdown
- 보조 자산: `templates/` (index, work-unit 템플릿), `examples/login-example.md`

### resume

PDF 이력서를 읽고 경력기술서 생성, 채용 트렌드 기반 평가, 프로젝트 코드 기반 보강, 자동 개선 루프까지 수행. JD URL/텍스트 와 매칭도 가능.

- 인자: `<pdf-path>` / `review <pdf-path>` / `improve <pdf-path>` / `enrich [target-score]` / `match <pdf-path> <jd-url-or-text>`

### jean:*

Claude Code 자체를 다루는 메타 도구 모음 — 스킬·에이전트·룰·훅·CLAUDE.md 를 만들고 다듬고 평가하는 생성기와, 코드베이스를 분석·정리하는 유틸리티로 구성. `.claude/commands/jean:*.md` 11개 커맨드 + `.claude/skills/jean-{analyze,deep-analyze,skill-creator}/` 보조 자산.

#### 오케스트레이터

##### `/jean:skill`

무엇이든 만들 때 가장 먼저 호출하는 통합 오케스트레이터. 요구사항을 분석해 `skill-creator` / `rule-creator` / `hook-creator` / `claude-md-creator` / `agent-creator` 중 필요한 도구를 자동 선택·순서 결정해 실행. 스킬 생성 시 eval pass_rate ≥ 0.95 도달까지 자동 루프, 모든 수정 시 사이드 이펙트 자동 검사.

- 인자: `{만들고 싶은 기능/행동 설명 or 리팩터링할 스킬 경로}`
- 트리거: "만들어줘", "추가해줘", "이 스킬 정리해줘", "자동화해줘", "어떤 프리미티브에 넣어야 해"

#### 프리미티브 생성기

`jean:skill` 이 위임하는 도구. 단독 호출도 가능.

##### `/jean:skill-creator`

새 Claude 스킬을 만들거나 기존 스킬을 개선·평가. Anthropic 공식 prompting best practices (XML 태그, 긍정 프레이밍, few-shot 3원칙, 골든룰)를 가이드에 통합. eval 루프로 description triggering 정확도까지 측정.

- 인자: `{스킬 이름 or 설명, 비워두면 대화로 진행}`

##### `/jean:agent-creator`

Claude Code 에이전트(`.claude/agents/<name>.md`)를 설계·작성·검증. Skill 과 달리 에이전트는 오케스트레이터가 위임하는 격리 전문가 — 모델 선택, 도구 권한 최소화, 입출력 계약 설계에 집중.

- 인자: `{에이전트 역할/이름 설명, 비워두면 대화로 진행}`
- 트리거: "서브에이전트 설계", "전문가 위임 구조", "병렬 처리용 에이전트"

##### `/jean:rule-creator`

기존 스킬을 분석해 rule (`CLAUDE.md` 또는 `.claude/rules/*.md`) 로 옮기면 좋은 부분을 찾아 제안·생성·정리. 기존 rule 과 중복/모순 자동 검출. 스킬에서 **추출**하는 도구이므로 대상 스킬이 먼저 존재해야 한다.

- 인자: `{스캔 대상 경로 or 비워서 기본 스캔}`

##### `/jean:hook-creator`

기존 스킬을 분석해 hook (`settings.json` 또는 `SKILL.md` frontmatter) 로 옮기면 좋은 결정적 동작을 찾아 제안·생성·검증·정리. dry-run 검증과 단계적 설치(local-first)로 안전성 확보.

- 인자: `{스캔 대상 경로 or 비워서 기본 스캔}`

##### `/jean:claude-md-creator`

프로젝트를 분석해 Claude Code 최적화 `CLAUDE.md` 를 생성하거나 기존 파일을 개선. 팀 컨벤션 정의, 신규 프로젝트 셋업, 기존 파일 정리 모두 커버.

- 인자: `{프로젝트 경로 or 비워서 현재 디렉토리}`
- 트리거: "CLAUDE.md 만들어줘", "프로젝트 컨텍스트 주입"

#### 정합성 도구

##### `/jean:skill-sync`

skills/agents/rules 파일을 수정·추가한 뒤 일관성 자동 점검·정리. 포맷 준수, 스킬 간 상호작용 계약, rules paths 불일치, 잔제 패턴, 유령 참조, 중복 내용을 탐지·수정. 단독 실행하거나 `jean:skill` 완료 후 자동 호출.

- 인자: `{파일명 or 비워서 전체 스캔}`

#### 코드베이스 분석

##### `/jean:analyze`

프로젝트 코드베이스 전체를 자동 스캔해 6개 영역 리포트(아키텍처·기능·워크플로우·데이터·건강도·온보딩)를 `docs/analysis/` 에 생성. 스택 자동 감지(Python · Node · Go · Rust · Java/Kotlin · Ruby) + 프레임워크별 패턴 추출.

- 인자: `[all|architecture|features|workflow|data|health|onboarding]` (기본: `all`)
- 트리거: "이 프로젝트 어떻게 돌아가", "아키텍처 감사", "온보딩 가이드"

##### `/jean:deep-analyze`

주제 자유형 심층 분석. 사용자가 지정한 이슈(N+1 쿼리, 중복 코드, dead code, 보안 취약점, 성능 핫스팟 등)에 대해 코드 전수 조사 후 파일:라인 단위 상세 리포트 생성. 검증 루프(최대 3회)로 false positive 제거.

- 인자: `{분석 주제 — 자유 형식}`
- 트리거: "어디서 발생하는지", "찾아줘", "누락된 곳", "감사", "이슈 추적"

#### 사고 정리

##### `/jean:idea`

두서없는 생각·아이디어를 대화로 함께 다듬어 LLM 이 효율적으로 실행할 수 있는 구조화된 명령으로 변환. 주제 무관 — 기술 구현, 설계 고민, 리서치, 비즈니스 기획, 문제 해결 모두 지원.

- 인자: `{당신의 생각 — 자유 형식}`
- 트리거: "구체화", "정리해줘", "brainstorm", "막연"

##### `/jean:tick-tock`

느슨한 수정 논의(대화, 코멘트, 생각 덩어리)를 어떤 AI 실행자에도 바로 붙여넣을 수 있는 명확한 수정 브리핑으로 압축. 수정 지시가 모호하거나 흩어져 있을 때 사용. 독립 스킬 — jean 생태계 오케스트레이터와는 연동하지 않음.

- 인자: `{수정 논의 내용 또는 수정 지시 — 자유 텍스트}`

## 라이선스

`jean:skill-creator` 는 Anthropic Skill Creator 기반 작업물을 포함하며 해당 LICENSE 는 `.claude/skills/jean-skill-creator/LICENSE.txt` 에 포함된다. 그 외 자체 작성 스킬은 별도 명시 전까지 개인 사용 목적.
