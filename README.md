# python-clean-arch

A Claude Code skill that guides Python API development using Clean Architecture principles.

Provides opinionated, ready-to-use conventions for projects built with **FastAPI · SQLAlchemy (async) · Dishka (IoC) · Dynaconf · Alembic · Pydantic v2**.

## What this skill does

When active, Claude will:

- **Scaffold new features** following a strict 4-layer structure: `domain → application → infrastructure → presentation`
- **Generate boilerplate** for entities, abstract repositories, ORM models, use cases, Pydantic schemas, and FastAPI routers — in the correct order and with the correct naming conventions
- **Enforce architectural boundaries** — dependencies always point inward; domain never imports from other layers
- **Bootstrap new projects** from scratch, including `pyproject.toml`, Alembic async setup, Dishka container wiring, and Dynaconf configuration
- **Answer architecture questions** about layer responsibilities, naming, IoC patterns, and test structure

## Stack

| Concern | Library |
|---|---|
| HTTP framework | FastAPI |
| ORM | SQLAlchemy (async) |
| Dependency injection | Dishka |
| Configuration | Dynaconf |
| Migrations | Alembic |
| Schemas / validation | Pydantic v2 |

## Layer structure

```
src/
├── domain/             # Pure business logic — no framework dependencies
│   └── <feature>/
│       ├── entities.py
│       └── repositories.py   # Abstract interfaces only
├── application/        # Use cases — orchestrates domain, no HTTP/DB knowledge
│   └── <feature>/
│       ├── use_cases.py
│       └── schemas.py
├── infrastructure/     # Concrete implementations (DB, external APIs)
│   └── <feature>/
│       ├── models.py
│       └── repositories.py
├── presentation/       # HTTP layer — FastAPI routers + request/response schemas
│   └── <feature>/
│       ├── router.py
│       └── schemas.py
├── shared/             # Cross-cutting: exceptions, base classes, utils
└── container.py        # Dishka IoC container
```

**Import direction rule (never violate):**
```
presentation → application → domain ← infrastructure
```

## Triggers

Claude applies this skill automatically when you mention use cases, repositories, entities, schemas, routers, IoC containers, or any layer-related question — even if "Clean Architecture" is not explicitly mentioned.

Also triggers for questions involving the stack libraries together (FastAPI + SQLAlchemy + Dishka + Dynaconf + Alembic + Pydantic v2), or Portuguese terms like _camada_, _use case_, _repositório_, _entidade_, _schema_, or _router_.

## Reference files

| File | Contents |
|---|---|
| `references/bootstrapping.md` | New project setup: skeleton, pyproject.toml, Alembic init, main.py |
| `references/dishka.md` | Full Dishka provider patterns, scopes, testing |
| `references/dynaconf.md` | Dynaconf config patterns, environments, secrets |
| `references/alembic.md` | Alembic async setup and migration conventions |
| `references/testing.md` | Test structure, mocking repos, async test patterns |
