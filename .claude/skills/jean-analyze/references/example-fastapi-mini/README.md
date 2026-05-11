# Mini Fixture: FastAPI Todo API

`/jean:analyze` 가 *어떤 결과를 만들어야 하는지* 보여주는 worked example.

이 디렉토리의 분석 결과 3개 (`01-architecture.md`, `02-feature-map.md`, `06-onboarding.md`) 는 **아래 가상 fixture 프로젝트** 를 분석한 결과다. 본문 (`jean:analyze.md`) 의 작업 지침이 실제로 적용된 모습 = 모델이 톤·디테일·구조를 학습하는 reference.

> 주의: 이 fixture 의 코드는 *교육 목적의 가상 코드* 다. 실제 분석 시에는 사용자의 진짜 코드베이스를 Glob/Grep/Read 로 직접 수집한다.

## Fixture 구조

```
fastapi-todo-mini/
├── pyproject.toml          # poetry, fastapi + sqlalchemy + asyncpg
├── poetry.lock
├── app/
│   ├── main.py             # FastAPI 진입점, /todos 라우터 등록
│   ├── config.py           # Settings (Pydantic BaseSettings)
│   ├── db.py               # SQLAlchemy AsyncEngine + Session
│   ├── api/
│   │   └── todos.py        # /todos CRUD 라우터 (5 endpoints)
│   ├── services/
│   │   └── todo_service.py # 비즈니스 로직
│   └── models/
│       └── todo.py         # Todo entity (SQLAlchemy)
└── tests/
    └── test_todos.py
```

## Fixture 코드 (압축)

`app/main.py`:
```python
from fastapi import FastAPI
from app.api.todos import router as todos_router
from app.config import settings

app = FastAPI(title=settings.APP_NAME)
app.include_router(todos_router, prefix="/todos", tags=["todos"])
```

`app/config.py`:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "Todo API"
    DATABASE_URL: str
    LOG_LEVEL: str = "INFO"

    class Config:
        env_file = ".env"

settings = Settings()
```

`app/db.py`:
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from app.config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=False)
async def get_session() -> AsyncSession: ...
```

`app/models/todo.py`:
```python
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import Integer, String, Boolean, DateTime
from datetime import datetime

class Todo(Base):
    __tablename__ = "todos"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(String(2000), nullable=True)
    completed: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

`app/api/todos.py`:
```python
from fastapi import APIRouter, Depends
from app.services.todo_service import TodoService

router = APIRouter()

@router.get("/")
async def list_todos(svc: TodoService = Depends()): ...

@router.post("/")
async def create_todo(payload: TodoCreate, svc: TodoService = Depends()): ...

@router.get("/{todo_id}")
async def get_todo(todo_id: int, svc: TodoService = Depends()): ...

@router.put("/{todo_id}")
async def update_todo(todo_id: int, payload: TodoUpdate, svc: TodoService = Depends()): ...

@router.delete("/{todo_id}")
async def delete_todo(todo_id: int, svc: TodoService = Depends()): ...
```

`app/services/todo_service.py`:
```python
class TodoService:
    def __init__(self, session: AsyncSession = Depends(get_session)):
        self.session = session

    async def list(self) -> list[Todo]: ...
    async def create(self, data: TodoCreate) -> Todo: ...
    async def get(self, todo_id: int) -> Todo: ...
    async def update(self, todo_id: int, data: TodoUpdate) -> Todo: ...
    async def delete(self, todo_id: int) -> None: ...
```

## 사용 (skill 본문에서)

`jean:analyze.md` 본문에서 이 디렉토리를 인용해 "이런 구조의 입력에 이런 출력을 기대" 라는 패턴을 보여준다. 모델은 메타데이터로 example 의 형태를 학습하고, 실제 분석 시 사용자 프로젝트의 디테일을 그 형태에 맞춰 채운다.
