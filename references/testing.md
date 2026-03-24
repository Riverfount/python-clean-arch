# Testing — Async Clean Architecture Reference

## Stack

```toml
pytest = "^8.0"
pytest-asyncio = "^0.23"
httpx = "^0.27"         # async test client for FastAPI
pytest-mock = "^3.14"
```

## pytest.ini / pyproject.toml

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

## Directory Structure

```
tests/
├── unit/
│   ├── domain/
│   │   └── test_todo_entity.py
│   └── application/
│       └── test_create_todo_use_case.py
├── integration/
│   └── infrastructure/
│       └── test_todo_repository.py
└── e2e/
    └── test_todo_router.py
```

---

## Unit Tests — Use Cases

Mock the repository; test only use-case logic.

```python
# tests/unit/application/test_create_todo_use_case.py
import pytest
from unittest.mock import AsyncMock
from domain.todo.entities import Todo
from application.todo.use_cases import CreateTodoUseCase
from application.todo.schemas import CreateTodoSchema


@pytest.fixture
def mock_repo():
    repo = AsyncMock()
    repo.save.return_value = Todo(id=1, title="Buy milk", done=False)
    return repo


@pytest.fixture
def use_case(mock_repo):
    return CreateTodoUseCase(repository=mock_repo)


async def test_create_todo_returns_schema(use_case, mock_repo):
    data = CreateTodoSchema(title="Buy milk")
    result = await use_case.execute(data)

    mock_repo.save.assert_called_once()
    assert result.id == 1
    assert result.title == "Buy milk"
    assert result.done is False
```

---

## Integration Tests — Repository

Use a real test DB (SQLite in-memory or test Postgres).

```python
# tests/integration/infrastructure/test_todo_repository.py
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from shared.base_model import Base
from infrastructure.todo.models import TodoModel  # noqa — needed for metadata
from infrastructure.todo.repositories import TodoRepositoryImpl
from domain.todo.entities import Todo


@pytest.fixture(scope="function")
async def session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    factory = async_sessionmaker(engine, expire_on_commit=False)
    async with factory() as s:
        yield s
    await engine.dispose()


async def test_save_and_get_todo(session: AsyncSession):
    repo = TodoRepositoryImpl(session)
    todo = Todo(id=None, title="Test", done=False)
    saved = await repo.save(todo)

    assert saved.id is not None
    fetched = await repo.get_by_id(saved.id)
    assert fetched.title == "Test"
```

---

## E2E Tests — FastAPI Router

Use `AsyncClient` from httpx + override Dishka container.

```python
# tests/e2e/test_todo_router.py
import pytest
from httpx import AsyncClient, ASGITransport
from unittest.mock import AsyncMock
from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka
from fastapi import FastAPI

from domain.todo.entities import Todo
from application.todo.use_cases import CreateTodoUseCase
from presentation.todo.router import todo_router
from tests.mocks.providers import MockTodoProvider  # your mock provider


@pytest.fixture
def app():
    app = FastAPI()
    container = make_async_container(MockTodoProvider())
    setup_dishka(container, app)
    app.include_router(todo_router)
    return app


@pytest.fixture
async def client(app):
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c


async def test_create_todo(client: AsyncClient):
    response = await client.post("/todos/", json={"title": "Buy milk"})
    assert response.status_code == 201
    assert response.json()["title"] == "Buy milk"
```

---

## Shared Mock Provider

```python
# tests/mocks/providers.py
from dishka import Provider, Scope, provide
from unittest.mock import AsyncMock
from domain.todo.entities import Todo
from domain.todo.repositories import TodoRepository
from application.todo.use_cases import CreateTodoUseCase


class MockTodoProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_todo_repository(self) -> TodoRepository:
        mock = AsyncMock(spec=TodoRepository)
        mock.save.return_value = Todo(id=1, title="Buy milk", done=False)
        return mock

    @provide(scope=Scope.REQUEST)
    def get_create_use_case(self, repo: TodoRepository) -> CreateTodoUseCase:
        return CreateTodoUseCase(repo)
```
