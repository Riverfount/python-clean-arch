# Alembic — Async Migrations Reference

## Setup

```toml
# pyproject.toml
alembic = "^1.13"
asyncpg = "^0.29"
```

```bash
alembic init -t async alembic
```

## alembic/env.py (async pattern)

```python
import asyncio
from logging.config import fileConfig
from sqlalchemy.ext.asyncio import async_engine_from_config
from sqlalchemy import pool
from alembic import context

from config import settings
from shared.base_model import Base
# Import ALL models so Alembic can detect them
import infrastructure.todo.models  # noqa
import infrastructure.user.models  # noqa

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

## shared/base_model.py

```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

## Common Commands

```bash
# Generate migration (auto-detect from models)
alembic revision --autogenerate -m "create todos table"

# Apply all pending migrations
alembic upgrade head

# Rollback one
alembic downgrade -1

# Show current revision
alembic current

# Show history
alembic history --verbose
```

## Migration File Convention

```python
# alembic/versions/20240315_create_todos.py
"""create todos table

Revision ID: abc123
Revises: 
Create Date: 2024-03-15
"""
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    op.create_table(
        "todos",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("title", sa.String(255), nullable=False),
        sa.Column("done", sa.Boolean, default=False, nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

def downgrade() -> None:
    op.drop_table("todos")
```

## Important: Register Models

Every time a new `infrastructure/<feature>/models.py` is created, add an import in `alembic/env.py`:
```python
import infrastructure.<feature>.models  # noqa
```
Without this, `--autogenerate` won't detect the new tables.
