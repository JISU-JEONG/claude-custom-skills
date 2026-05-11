# 아키텍처 분석

> 생성일: 2026-04-26 12:00 | 프로젝트: fastapi-todo-mini

## TL;DR

- **스택**: Python 3.12 + FastAPI 0.110 + SQLAlchemy 2.0 (async) + PostgreSQL + Pydantic v2
- **레이어**: API (`app/api/`) → Service (`app/services/`) → Model (`app/models/`) 3-tier
- **DI**: FastAPI `Depends` 기반 (별도 컨테이너 없음)
- **인증**: 없음 (미감지)
- **DB**: PostgreSQL via SQLAlchemy AsyncEngine

## 1. 레이어 구조

### 1.1 API 레이어 (`app/api/`)

| 파일 | 클래스/함수 | 역할 | 의존 대상 |
|------|------------|------|----------|
| `app/api/todos.py` | `router` (APIRouter) | `/todos` 5개 엔드포인트 | `TodoService` |
| `app/api/todos.py` | `list_todos` | GET / 핸들러 | `TodoService.list()` |
| `app/api/todos.py` | `create_todo` | POST / 핸들러 | `TodoService.create()` |
| `app/api/todos.py` | `get_todo` | GET /{id} 핸들러 | `TodoService.get()` |
| `app/api/todos.py` | `update_todo` | PUT /{id} 핸들러 | `TodoService.update()` |
| `app/api/todos.py` | `delete_todo` | DELETE /{id} 핸들러 | `TodoService.delete()` |

### 1.2 Service 레이어 (`app/services/`)

| 파일 | 클래스/함수 | 역할 | 의존 대상 |
|------|------------|------|----------|
| `app/services/todo_service.py` | `TodoService` | Todo 비즈니스 로직 (CRUD 5개 메서드) | `AsyncSession`, `Todo` 모델 |

### 1.3 Model 레이어 (`app/models/`)

| 파일 | 클래스/함수 | 역할 | 의존 대상 |
|------|------------|------|----------|
| `app/models/todo.py` | `Todo` | SQLAlchemy ORM 엔티티 (`todos` 테이블) | `Base` (declarative) |

### 1.4 인프라 (`app/db.py`, `app/config.py`)

| 파일 | 클래스/함수 | 역할 |
|------|------------|------|
| `app/db.py` | `engine`, `get_session` | AsyncEngine, 세션 팩토리 |
| `app/config.py` | `Settings`, `settings` | 환경변수 로딩 (Pydantic BaseSettings) |

## 2. 의존성 구조

별도 DI 컨테이너 없음. FastAPI `Depends` 로 핸들러 → Service → Session 주입.

```
FastAPI Request
  ▼
list_todos / create_todo / ... [app/api/todos.py]
  ▼ Depends(TodoService)
TodoService [app/services/todo_service.py]
  ▼ Depends(get_session)
AsyncSession [app/db.py]
  ▼
PostgreSQL
```

## 3. 설정 관리

| 설정 항목 | 타입 | 용도 |
|-----------|------|------|
| `APP_NAME` | str | FastAPI title (기본값 "Todo API") |
| `DATABASE_URL` | str | PostgreSQL 연결 문자열 (필수) |
| `LOG_LEVEL` | str | 로깅 레벨 (기본값 "INFO") |

`.env` 파일 로드. 값은 변수명만 기재.

## 4. 예외/에러 처리

`exceptions/` 디렉토리 부재. FastAPI 의 기본 `HTTPException` 사용 추정 — `[확인 필요]`.

## 5. 인증/보안

미감지. 인증 미들웨어·의존성 없음.
