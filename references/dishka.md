# Dishka — IoC Container Reference

## Installation
```toml
# pyproject.toml
dishka = "^1.0"
```

## Scopes

| Scope | Lifetime | Use for |
|---|---|---|
| `Scope.APP` | Application lifetime | DB engine, config, external clients |
| `Scope.REQUEST` | Per HTTP request | `AsyncSession`, use cases, repositories |

## Provider Pattern

```python
# container.py
from dishka import Provider, Scope, provide, make_async_container
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from config import settings
from domain.todo.repositories import TodoRepository
from infrastructure.todo.repositories import TodoRepositoryImpl
from application.todo.use_cases import CreateTodoUseCase, GetTodoUseCase


class DatabaseProvider(Provider):
    @provide(scope=Scope.APP)
    async def get_engine(self):
        engine = create_async_engine(settings.DATABASE_URL, echo=settings.DB_ECHO)
        yield engine
        await engine.dispose()

    @provide(scope=Scope.APP)
    def get_session_factory(self, engine) -> async_sessionmaker[AsyncSession]:
        return async_sessionmaker(engine, expire_on_commit=False)

    @provide(scope=Scope.REQUEST)
    async def get_session(self, factory: async_sessionmaker[AsyncSession]) -> AsyncSession:
        async with factory() as session:
            yield session


class TodoProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_todo_repository(self, session: AsyncSession) -> TodoRepository:
        return TodoRepositoryImpl(session)

    @provide(scope=Scope.REQUEST)
    def get_create_use_case(self, repo: TodoRepository) -> CreateTodoUseCase:
        return CreateTodoUseCase(repo)

    @provide(scope=Scope.REQUEST)
    def get_get_use_case(self, repo: TodoRepository) -> GetTodoUseCase:
        return GetTodoUseCase(repo)


def create_container():
    return make_async_container(
        DatabaseProvider(),
        TodoProvider(),
    )
```

## FastAPI Integration

```python
# main.py
from fastapi import FastAPI
from dishka.integrations.fastapi import setup_dishka
from container import create_container
from presentation.todo.router import todo_router

app = FastAPI()
container = create_container()
setup_dishka(container, app)

app.include_router(todo_router)
```

## Injecting in Routers

```python
from dishka.integrations.fastapi import FromDishka, inject

@router.post("/")
@inject  # Required decorator
async def create(
    body: TodoCreateRequest,
    use_case: FromDishka[CreateTodoUseCase],
) -> TodoResponse:
    ...
```

## Injecting in Background Tasks / non-HTTP

```python
from dishka import AsyncContainer

async def my_task(container: AsyncContainer):
    async with container() as request_container:
        use_case = await request_container.get(SomeUseCase)
        await use_case.execute(...)
```

## Testing with Dishka

```python
# Override providers for tests
from dishka import make_async_container
from unittest.mock import AsyncMock

class MockTodoProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_todo_repository(self) -> TodoRepository:
        mock = AsyncMock(spec=TodoRepository)
        mock.save.return_value = Todo(id=1, title="Test", done=False)
        return mock

    @provide(scope=Scope.REQUEST)
    def get_create_use_case(self, repo: TodoRepository) -> CreateTodoUseCase:
        return CreateTodoUseCase(repo)

test_container = make_async_container(MockTodoProvider())
```
