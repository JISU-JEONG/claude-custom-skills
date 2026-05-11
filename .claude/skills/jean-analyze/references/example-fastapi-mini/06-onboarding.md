# 신입 온보딩 가이드

> 생성일: 2026-04-26 12:00
> 대상: 프로젝트에 처음 합류하는 개발자

## 프로젝트 한 줄 소개

Todo 항목을 PostgreSQL 에 저장·조회하는 5개 엔드포인트짜리 단순 FastAPI 서비스.

## 1. 로컬 환경 세팅

### 1.1 사전 요구사항

- Python 3.12+
- PostgreSQL 14+
- Poetry 1.7+

### 1.2 설치 순서

```bash
# 의존성 설치
poetry install

# 환경변수 (.env)
cat > .env <<EOF
DATABASE_URL=postgresql+asyncpg://localhost:5432/todos
APP_NAME=Todo API
LOG_LEVEL=INFO
EOF

# 마이그레이션 (alembic 사용 시)
poetry run alembic upgrade head

# 실행
poetry run uvicorn app.main:app --reload
```

## 2. 이것부터 읽어라 (순서대로)

| 순서 | 파일 | 읽는 이유 |
|------|------|----------|
| 1 | `app/main.py` | FastAPI 앱 진입점, 라우터 등록 흐름 |
| 2 | `app/config.py` | 어떤 환경변수가 필요한지 |
| 3 | `app/api/todos.py` | 5개 엔드포인트 명세 |
| 4 | `app/services/todo_service.py` | 비즈니스 로직 + DB 호출 패턴 |
| 5 | `app/models/todo.py` | 데이터 스키마 |
| 6 | `app/db.py` | 세션 / 엔진 설정 |
| 7 | `tests/test_todos.py` | 테스트 작성 패턴 |

## 3. 핵심 비즈니스 흐름 워크스루

### 3.1 Todo 생성 요청

1. 클라이언트가 `POST /todos/` 로 `{title, description}` 전송
2. FastAPI 가 Pydantic `TodoCreate` 로 입력 검증
3. `create_todo` 핸들러가 `TodoService` 의존성 주입 받음
4. `TodoService.create(data)` 가 `Todo` 인스턴스 생성 → `session.add` → `commit`
5. 반환된 ORM 객체를 Pydantic `TodoOut` 으로 직렬화 → 응답

### 3.2 Todo 조회 (단건)

1. `GET /todos/{todo_id}` 호출
2. `get_todo` 가 `TodoService.get(todo_id)` 호출
3. `session.get(Todo, todo_id)` 로 조회 → `None` 이면 404 반환
4. 직렬화 후 응답

## 4. 자주 하는 작업별 가이드

### 4.1 새 API 엔드포인트 추가

기존 패턴을 따른다:

1. `app/api/todos.py` (또는 새 도메인이면 `app/api/<domain>.py` 신설) 에 라우터 함수 추가
2. 비즈니스 로직은 `TodoService` 에 메서드 추가 (또는 새 Service 클래스)
3. 새 도메인이면 `app/main.py` 에 `app.include_router` 추가
4. 요청/응답 Pydantic 스키마는 라우터 파일 상단에 정의
5. `tests/` 에 테스트 파일 추가

### 4.2 데이터 모델 필드 추가

1. `app/models/todo.py` 의 `Todo` 클래스에 `Mapped[T]` 컬럼 추가
2. `alembic revision --autogenerate -m "<설명>"` 으로 마이그레이션 생성
3. `alembic upgrade head` 적용
4. Pydantic 스키마 (`TodoCreate`/`TodoOut`) 에 해당 필드 반영

## 5. 디버깅 팁

| 상황 | 확인 방법 |
|------|----------|
| `DATABASE_URL` 연결 실패 | `.env` 의 `DATABASE_URL` 형식 확인 (`postgresql+asyncpg://...`) |
| 라우터가 안 잡힘 | `app/main.py` 의 `include_router` 호출 + prefix 확인 |
| 마이그레이션 충돌 | `alembic history` 로 분기 확인, 필요시 `merge` |
| 테스트 DB 격리 | `tests/conftest.py` 의 fixture 가 트랜잭션 롤백 사용 중인지 확인 |

## 6. 추가 리소스

- [아키텍처 분석](./01-architecture.md)
- [기능 매핑](./02-feature-map.md)
