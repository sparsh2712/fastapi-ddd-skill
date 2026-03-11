# Infrastructure Layer

---

## Design Rules

- Infrastructure is a detail — it exists to serve the domain, not define it
- Settings via `BaseSettings` — every infra component reads from env vars through typed settings classes
- Repositories live in the infrastructure layer — they receive connections or clients as constructor arguments, never creating their own

---

## Database Connection

The connection layer creates an async engine at startup and stores it on application state. The Unit of Work acquires connections from it per request.

```python
# infrastructure/postgresql/settings.py
from pydantic_settings import BaseSettings

class PostgresSettings(BaseSettings):
    POSTGRES_HOST: str
    POSTGRES_PORT: int = 5432
    POSTGRES_USER: str
    POSTGRES_PASSWORD: str
    POSTGRES_DATABASE: str
    POSTGRES_POOL_SIZE: int = 5
    POSTGRES_MAX_OVERFLOW: int = 10

    model_config = {"env_file": ".env", "extra": "ignore"}
```

```python
# infrastructure/database/connection.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncEngine
from infrastructure.postgresql.settings import PostgresSettings

def create_engine(settings: PostgresSettings) -> AsyncEngine:
    url = (
        f"postgresql+asyncpg://{settings.POSTGRES_USER}:{settings.POSTGRES_PASSWORD}"
        f"@{settings.POSTGRES_HOST}:{settings.POSTGRES_PORT}/{settings.POSTGRES_DATABASE}"
    )
    return create_async_engine(
        url,
        pool_size=settings.POSTGRES_POOL_SIZE,
        max_overflow=settings.POSTGRES_MAX_OVERFLOW,
    )
```

---

## Email

```python
# infrastructure/email/client.py
import typing
from dataclasses import dataclass
from pydantic import BaseModel

class EmailPayload(BaseModel):
    from_address: str
    to: list[str]
    cc: list[str] | None = None
    subject: str
    body_html: str
    body_text: str

@dataclass
class SendResult:
    success: bool
    message_id: str | None
    error: str | None

class EmailClientProtocol(typing.Protocol):
    async def send(self, email: EmailPayload) -> SendResult: ...
    async def send_bulk(self, emails: list[EmailPayload]) -> list[SendResult]: ...
```

---

## External Service Gateways

Any external dependency the domain needs is accessed through a gateway. The gateway is defined in the infrastructure layer and passed into services as a constructor argument or function parameter. The domain never calls external services directly.

```python
# infrastructure/gateways/inventory_gateway.py
import typing
from dataclasses import dataclass

@dataclass
class StockLevel:
    sku: str
    available: int
    location: str

class InventoryGatewayProtocol(typing.Protocol):
    async def get_stock_level(self, sku: str) -> StockLevel: ...
    async def reserve(self, sku: str, quantity: int) -> bool: ...
```

---

## Lifecycle Ownership

Infrastructure clients are created once at startup and stored on application state. They are disposed on shutdown.

```python
# entrypoints/app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from infrastructure.database.connection import create_engine
from infrastructure.email.factory import get_email_client
from config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.engine = create_engine(settings.postgres)
    app.state.email = get_email_client(settings.email)
    yield
    await app.state.engine.dispose()

app = FastAPI(lifespan=lifespan)
```

Nothing else manages client creation or disposal.
