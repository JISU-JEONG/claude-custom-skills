# 기능 매핑

> 생성일: 2026-04-26 12:00

## TL;DR

- 총 API 엔드포인트: 5개
- 도메인: Todo CRUD 단일 도메인

## 1. API 엔드포인트 전체 목록

| # | Method | Path | 핸들러 파일:함수 | 설명 |
|---|--------|------|-----------------|------|
| 1 | GET | `/todos/` | `app/api/todos.py:list_todos` | Todo 목록 조회 |
| 2 | POST | `/todos/` | `app/api/todos.py:create_todo` | Todo 생성 |
| 3 | GET | `/todos/{todo_id}` | `app/api/todos.py:get_todo` | Todo 단건 조회 |
| 4 | PUT | `/todos/{todo_id}` | `app/api/todos.py:update_todo` | Todo 수정 |
| 5 | DELETE | `/todos/{todo_id}` | `app/api/todos.py:delete_todo` | Todo 삭제 |

## 2. 기능별 파일 추적 맵

### 2.1 Todo 생성

**진입점**: `POST /todos/`

```
create_todo(payload: TodoCreate) [app/api/todos.py]
  ▼
TodoService.create(data: TodoCreate) [app/services/todo_service.py]
  ▼
session.add(Todo(**data.model_dump())) [app/models/todo.py]
  ▼
session.commit() → Todo 인스턴스 반환
  ▼
응답: TodoOut (Pydantic)
```

**관련 파일 목록**:

| 역할 | 파일 |
|------|------|
| 핸들러 | `app/api/todos.py` |
| 서비스 | `app/services/todo_service.py` |
| 모델 | `app/models/todo.py` |
| DB 세션 | `app/db.py` |
| 요청/응답 스키마 | `app/api/todos.py` 내 `TodoCreate`, `TodoOut` (Pydantic) |

### 2.2 Todo 조회 / 수정 / 삭제

각 엔드포인트의 흐름은 2.1 과 동일하게 `핸들러 → Service.{메서드} → Session 작업 → Pydantic 응답` 순. `get_todo`·`update_todo`·`delete_todo` 의 경우 `todo_id` PathParam 으로 단건 조회 후 처리.

## 3. 외부 API 연동 맵

해당 없음. `clients/`·`adapters/` 디렉토리 부재.
