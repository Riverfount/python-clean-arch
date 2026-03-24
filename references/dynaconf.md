# Dynaconf — Configuration Reference

## Installation
```toml
dynaconf = "^3.2"
```

## Project Setup

```
project/
├── config.py
├── settings.toml
└── .secrets.toml       # gitignored
```

### config.py
```python
from dynaconf import Dynaconf

settings = Dynaconf(
    envvar_prefix="APP",
    settings_files=["settings.toml", ".secrets.toml"],
    environments=True,
    load_dotenv=True,
)
```

### settings.toml
```toml
[default]
APP_NAME = "My API"
DEBUG = false
DB_ECHO = false

[development]
DEBUG = true
DB_ECHO = true
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/dev_db"

[production]
DEBUG = false
DATABASE_URL = "@format {env[DATABASE_URL]}"  # from env var
```

### .secrets.toml (gitignored)
```toml
[default]
SECRET_KEY = "change-me"

[development]
SECRET_KEY = "dev-secret-key"
```

### .gitignore
```
.secrets.toml
```

## Accessing Settings

```python
from config import settings

# Direct access
db_url = settings.DATABASE_URL
debug = settings.DEBUG

# With default fallback
timeout = settings.get("TIMEOUT", default=30)

# Nested
redis_host = settings.REDIS.HOST
```

## Environment Selection

```bash
# Via env var
ENV_FOR_DYNACONF=production python -m uvicorn main:app

# Or in .env
ENV_FOR_DYNACONF=development
```

## Validation (optional but recommended)

```python
# config.py
settings = Dynaconf(
    ...
    validators=[
        Validator("DATABASE_URL", must_exist=True),
        Validator("SECRET_KEY", must_exist=True),
        Validator("DEBUG", is_type_of=bool),
    ]
)

settings.validators.validate()  # Raises on startup if invalid
```

## Type Casting

Dynaconf auto-casts TOML types. For env vars:
```bash
APP_DEBUG=true         # → bool True
APP_PORT=8080          # → int 8080
APP_HOSTS='["a","b"]'  # → list
```

## Usage in Container

```python
# container.py
from config import settings

class DatabaseProvider(Provider):
    @provide(scope=Scope.APP)
    async def get_engine(self):
        engine = create_async_engine(
            settings.DATABASE_URL,
            echo=settings.DB_ECHO,
        )
        yield engine
        await engine.dispose()
```
