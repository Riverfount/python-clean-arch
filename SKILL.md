---
name: python-clean-arch
description: >
  Apply this skill whenever the user is working on Python projects using Clean Architecture,
  including: creating or scaffolding new features/modules, writing entities, use cases,
  repositories, schemas, routers, or IoC containers; reviewing or refactoring code for
  architectural correctness; generating boilerplate for domain/application/infrastructure/
  presentation layers; or asking about conventions, naming, or layer responsibilities.
  Also trigger for any question involving FastAPI + SQLAlchemy + Dishka + Dynaconf + Alembic
  + Pydantic v2 together, even if "Clean Architecture" is not explicitly mentioned.
  Use this skill proactively when the user mentions "camada", "use case", "repositório",
  "entidade", "schema", "router", "IoC", or any structural aspect of their Python API.
---

# Python Clean Architecture Skill

Stack padrão: **FastAPI · SQLAlchemy async · Dishka (IoC) · Dynaconf · Alembic · Pydantic v2**

For detailed reference on any section, read the corresponding file in `references/`.

---

## 1. Layer Structure

```
src/
├── domain/             # Pure business logic — no framework dependencies
│   └── <feature>/
│       ├── entities.py
│       └── repositories.py   # Abstract interfaces only
├── application/        # Use cases — orchestrates domain, no HTTP/DB knowledge
│   └── <feature>/
│       ├── use_cases.py
│       └── schemas.py        # Pydantic schemas for use-case I/O
├── infrastructure/     # Concrete implementations (DB, external APIs)
│   └── <feature>/
│       ├── models.py         # SQLAlchemy ORM models
│       ├── repositories.py   # Concrete repo implementations
│       └── migrations/       # Alembic (usually at project root)
├── presentation/       # HTTP layer — FastAPI routers + request/response schemas
│   └── <feature>/
│       ├── router.py
│       └── schemas.py        # Pydantic HTTP schemas
├── shared/             # Cross-cutting: exceptions, base classes, utils
│   ├── exceptions.py
│   └── base_repository.py
└── container.py        # Dishka IoC container (all providers)
```

> **Rule:** dependencies point inward only.  
> `presentation → application → domain ← infrastructure`

---

## 2. Naming Conventions

### Schemas (Pydantic)

| Context | Suffix | Example |
|---|---|---|
| HTTP create body | `CreateRequest` | `TodoCreateRequest` |
| HTTP update body | `UpdateRequest` | `TodoUpdateRequest` |
| HTTP response | `Response` | `TodoResponse` |
| Use-case base / shared | `Schema` | `TodoSchema` |

- `presentation/schemas.py` → `CreateRequest`, `UpdateRequest`, `Response`  
- `application/schemas.py` → `Schema` (internal use-case data contracts)

### Other Identifiers

| Component | Pattern | Example |
|---|---|---|
| Entity | `<Name>` (plain class or dataclass) | `Todo` |
| ORM Model | `<Name>Model` | `TodoModel` |
| Abstract Repo | `<Name>Repository` (ABC) | `TodoRepository` |
| Concrete Repo | `<Name>RepositoryImpl` | `TodoRepositoryImpl` |
| Use Case | `<Verb><Name>UseCase` | `CreateTodoUseCase` |
| Router | `<name>_router` | `todo_router` |

---

## 3. Layer Responsibilities (quick ref)

**`domain/`**  
- Entities: plain Python classes or dataclasses. No Pydantic, no SQLAlchemy.  
- Repository interfaces: `ABC` with `async` abstract methods. Return domain entities.

**`application/`**  
- One class per use case. Constructor receives repo interfaces (injected by Dishka).  
- Single public method (usually `execute()`).  
- Receives/returns `application/schemas.py` types, never HTTP schemas or ORM models.

**`infrastructure/`**  
- SQLAlchemy `async` session via `AsyncSession`.  
- Repo implementations convert between ORM models ↔ domain entities.  
- Never import from `presentation/`.

**`presentation/`**  
- FastAPI router. Receives HTTP schemas, calls use cases, returns HTTP schemas.  
- No business logic. No direct DB access.

**`shared/`**  
- Base exceptions (`DomainException`, `NotFoundError`, etc.).  
- Optional base repository class, generic types, utilities.

---

## 4. IoC — Dishka

Read `references/dishka.md` for full provider patterns.

Quick rules:
- All providers live in `container.py` (or `providers/` for large projects).
- Use `provide` decorator with correct `Scope` (`APP`, `REQUEST`).
- Inject into use cases via constructor; inject into routers via `FromDishka` dependency.
- `AsyncSession` is provided with `REQUEST` scope.

---

## 5. Configuration — Dynaconf

Read `references/dynaconf.md` for full config patterns.

Quick rules:
- Settings file: `settings.toml` + `.secrets.toml` (gitignored).
- Access via `from config import settings`.
- Environments: `default`, `development`, `production`.
- Never hardcode secrets; always use `settings.KEY`.

---

## 6. Code Generation Checklist

When creating a new feature `<name>`, generate in order:

1. `domain/<name>/entities.py` — Entity class  
2. `domain/<name>/repositories.py` — Abstract repository (ABC)  
3. `infrastructure/<name>/models.py` — SQLAlchemy ORM model  
4. `infrastructure/<name>/repositories.py` — Concrete repository  
5. `application/<name>/schemas.py` — Application-level schemas  
6. `application/<name>/use_cases.py` — One class per use case  
7. `presentation/<name>/schemas.py` — HTTP Request/Response schemas  
8. `presentation/<name>/router.py` — FastAPI router  
9. `container.py` — Register new providers  

---

## 7. Key Patterns (inline examples)

### Entity
```python
# domain/todo/entities.py
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Todo:
    id: int | None
    title: str
    done: bool
    created_at: datetime
```

### Abstract Repository
```python
# domain/todo/repositories.py
from abc import ABC, abstractmethod
from .entities import Todo

class TodoRepository(ABC):
    @abstractmethod
    async def get_by_id(self, todo_id: int) -> Todo | None: ...

    @abstractmethod
    async def save(self, todo: Todo) -> Todo: ...
```

### ORM Model
```python
# infrastructure/todo/models.py
from sqlalchemy.orm import Mapped, mapped_column
from shared.base_model import Base

class TodoModel(Base):
    __tablename__ = "todos"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    done: Mapped[bool] = mapped_column(default=False)
```

### Use Case
```python
# application/todo/use_cases.py
from domain.todo.repositories import TodoRepository
from domain.todo.entities import Todo
from .schemas import CreateTodoSchema, TodoSchema

class CreateTodoUseCase:
    def __init__(self, repository: TodoRepository) -> None:
        self._repository = repository

    async def execute(self, data: CreateTodoSchema) -> TodoSchema:
        todo = Todo(id=None, title=data.title, done=False, ...)
        saved = await self._repository.save(todo)
        return TodoSchema.model_validate(saved)
```

### Presentation Schemas
```python
# presentation/todo/schemas.py
from pydantic import BaseModel

class TodoCreateRequest(BaseModel):
    title: str

class TodoUpdateRequest(BaseModel):
    title: str | None = None
    done: bool | None = None

class TodoResponse(BaseModel):
    id: int
    title: str
    done: bool

    model_config = {"from_attributes": True}
```

### Router
```python
# presentation/todo/router.py
from fastapi import APIRouter
from dishka.integrations.fastapi import FromDishka, inject
from application.todo.use_cases import CreateTodoUseCase
from .schemas import TodoCreateRequest, TodoResponse

todo_router = APIRouter(prefix="/todos", tags=["todos"])

@todo_router.post("/", response_model=TodoResponse, status_code=201)
@inject
async def create_todo(
    body: TodoCreateRequest,
    use_case: FromDishka[CreateTodoUseCase],
) -> TodoResponse:
    result = await use_case.execute(body)
    return TodoResponse.model_validate(result)
```

---

## 8. New Project Bootstrapping

Read `references/bootstrapping.md` for the full step-by-step guide.

Quick order of operations:
1. Create directory skeleton (`src/` + all layer dirs)
2. Set up `settings.toml`, `pyproject.toml`, `shared/base_model.py`
3. Initialize Alembic (`alembic init -t async alembic`)
4. Set up `main.py` + `container.py` (empty providers)
5. For each feature: follow the 9-step code generation checklist (Section 6)

**Import direction rule** (never violate):
```
presentation → application → domain ← infrastructure
```
`domain` never imports from any other layer. `infrastructure` never imports from `presentation`.

---

## Reference Files

- `references/bootstrapping.md` — New project setup: skeleton, pyproject.toml, Alembic init, main.py
- `references/dishka.md` — Full Dishka provider patterns, scopes, testing
- `references/dynaconf.md` — Dynaconf config patterns, environments, secrets
- `references/alembic.md` — Alembic async setup and migration conventions
- `references/testing.md` — Test structure, mocking repos, async test patterns
