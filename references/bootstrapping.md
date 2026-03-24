# New Project Bootstrapping

## 1. Directory Skeleton

```bash
mkdir -p src/{domain,application,infrastructure,presentation,shared}
touch src/__init__.py src/main.py src/container.py
touch settings.toml .secrets.toml .env
echo ".secrets.toml" >> .gitignore
echo ".env" >> .gitignore
```

For each new feature:
```bash
feature=todo  # change as needed
mkdir -p src/{domain,application,infrastructure,presentation}/$feature
touch src/{domain,application,infrastructure,presentation}/$feature/__init__.py
```

---

## 2. Inicializar o projeto com UV

```bash
uv init myapp
cd myapp
uv python pin 3.12
```

Adicionar dependências de produção:
```bash
uv add fastapi "uvicorn[standard]" "sqlalchemy[asyncio]" asyncpg alembic dishka dynaconf pydantic
```

Adicionar dependências de desenvolvimento como grupo:
```bash
uv add --group dev pytest pytest-asyncio httpx aiosqlite pytest-mock ruff
```

O `pyproject.toml` resultante terá `[dependency-groups]` no lugar de `[project.optional-dependencies]`.
Adicione manualmente as seções de configuração de ferramentas:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
ignore = ["E501"]

[tool.ruff.lint.isort]
known-first-party = ["src"]
```

---

## 3. settings.toml + .secrets.toml

```toml
# settings.toml
[default]
APP_NAME = "MyApp"
DEBUG = false
DB_ECHO = false

[development]
DEBUG = true
DB_ECHO = true
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/myapp_dev"

[production]
DEBUG = false
# DATABASE_URL comes from env var in production
```

```toml
# .secrets.toml  (gitignored)
[default]
SECRET_KEY = "change-me-in-production"

[development]
SECRET_KEY = "dev-only-secret"
```

```python
# src/config.py
from dynaconf import Dynaconf, Validator

settings = Dynaconf(
    envvar_prefix="APP",
    settings_files=["settings.toml", ".secrets.toml"],
    environments=True,
    load_dotenv=True,
    validators=[
        Validator("DATABASE_URL", must_exist=True),
        Validator("SECRET_KEY", must_exist=True),
    ],
)
```

---

## 4. Shared Base

```python
# src/shared/base_model.py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

```python
# src/shared/exceptions.py
class DomainException(Exception):
    """Base for all domain exceptions."""

class NotFoundError(DomainException):
    def __init__(self, entity: str, id: int | str):
        super().__init__(f"{entity} with id={id} not found")

class ConflictError(DomainException):
    pass
```

---

## 5. Alembic Init (async)

```bash
alembic init -t async alembic
```

Edit `alembic/env.py` — replace the content with the async pattern from `references/alembic.md`.

Key points:
- Point `target_metadata` to `Base.metadata` from `shared/base_model.py`
- Set `sqlalchemy.url` from `settings.DATABASE_URL`
- **Add an import for every new ORM model** — without this, `--autogenerate` won't detect new tables

```python
# At the top of alembic/env.py — add one line per new feature
import src.infrastructure.todo.models   # noqa
import src.infrastructure.user.models   # noqa
```

---

## 6. main.py + container.py

```python
# src/main.py
from fastapi import FastAPI
from dishka.integrations.fastapi import setup_dishka
from container import create_container

def create_app() -> FastAPI:
    app = FastAPI(title="MyApp")
    setup_dishka(create_container(), app)

    # Register routers as features are added:
    # from presentation.todo.router import todo_router
    # app.include_router(todo_router, prefix="/api/v1")

    return app

app = create_app()
```

```python
# src/container.py
from dishka import make_async_container
from dishka.integrations.fastapi import FastAPIProvider

# Import and add providers as features are created:
# from infrastructure.todo.providers import TodoProvider

def create_container():
    return make_async_container(
        FastAPIProvider(),
        # TodoProvider(),
    )
```

---

## 7. Feature Creation Order (repeat for each feature)

Always build from domain outward — each layer depends only on the one below it:

```
1. domain/<f>/entities.py           ← dataclass, no external deps
2. domain/<f>/repositories.py       ← ABC interface, async methods
3. infrastructure/<f>/models.py     ← SQLAlchemy ORM model
4. infrastructure/<f>/repositories.py ← concrete impl, maps model ↔ entity
5. application/<f>/schemas.py       ← Pydantic or dataclass for use-case I/O
6. application/<f>/use_cases.py     ← one class per use case
7. presentation/<f>/schemas.py      ← CreateRequest / UpdateRequest / Response
8. presentation/<f>/router.py       ← FastAPI router, @inject
9. container.py                     ← add new providers
   alembic/env.py                   ← add import for new ORM model
   main.py                          ← include_router
```

---

## 8. Import Direction — Never Violate

```
presentation → application → domain ← infrastructure
```

| Import | Allowed? |
|---|---|
| `presentation` → `application` | ✅ |
| `application` → `domain` | ✅ |
| `infrastructure` → `domain` | ✅ |
| `domain` → anything | ❌ |
| `infrastructure` → `presentation` | ❌ |
| `infrastructure` → `application` | ❌ |
| `presentation` → `infrastructure` | ❌ |

---

## 9. Running Locally

```bash
# Sincronizar ambiente (produção + grupo dev)
uv sync --group dev

# Set environment
export ENV_FOR_DYNACONF=development

# Lint e format
uv run ruff check src/
uv run ruff format src/

# Run migrations
uv run alembic upgrade head

# Start server
uv run uvicorn src.main:app --reload

# Testes
uv run pytest
```
