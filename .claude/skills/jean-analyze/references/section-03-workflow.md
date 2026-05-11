# Section 03: 워크플로우 / 파이프라인 분석 (`03-workflow.md`)

**생성 조건**: `has_workflow = true` 일 때만 (`graph/`, `workflows/`, `pipelines/`, `agents/` 디렉토리 존재 시)

## 분석 순서

1. **워크플로우 정의 파일 Glob** — 프레임워크별 패턴.
   - LangGraph: `graph/graphs/*.py` → `add_node`, `add_edge` 패턴
   - LangChain: `chains/`, `agents/` → 체인 / 에이전트 정의
   - Celery: `tasks/` → `@task`, `@shared_task` 패턴
   - Temporal / Airflow: workflow 정의 파일
   - 커스텀: `pipelines/`, `workflows/` 디렉토리 탐색
2. **노드 / 스텝 / 태스크 파일 Read** — 이름·역할·의존 관계를 추출한다.
3. **State / Context 스키마 파일 Read** — 필드 목록을 수집한다.
4. **Tool / Action 정의 파일 Read** — 도구 목록을 수집한다.

## 출력 포맷

<output_format>
```markdown
# 워크플로우 분석

> 생성일: {YYYY-MM-DD HH:MM}

## TL;DR

- 워크플로우 엔진: {LangGraph|Celery|Airflow|커스텀|...}
- 워크플로우: {N}개
- 노드/스텝: {N}개
- Tool/Action: {N}개

## 1. {워크플로우명}

### 1.1 노드/스텝 목록

| 이름 | 파일 | 역할 |
|------|------|------|
| {실제 코드에서 추출} | ... | ... |

### 1.2 흐름도

```
START → {첫 번째} → {분기} → ... → END
```

### 1.3 State/Context 스키마 (해당 시)

| 필드 | 타입 | 용도 |
|------|------|------|
| ... | ... | ... |

## N. Tool/Action 목록 (해당 시)

| 이름 | 파일 | 용도 |
|------|------|------|
| ... | ... | ... |
```
</output_format>
