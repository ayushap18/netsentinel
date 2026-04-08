# Phase 1: Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Core server running with FastAPI, Docker Compose orchestrating TimescaleDB + Redis, database schema applied, basic agent connecting via WebSocket with heartbeats, packet capture on laptop via Scapy, packets stored in TimescaleDB, and queryable via REST API.

**Architecture:** Hub-and-spoke model — a FastAPI Core server receives WebSocket connections from device agents, processes incoming packet data through an L1 Packet Engine, stores in TimescaleDB (time-series) and Redis (real-time state), and exposes REST endpoints. Agents run as standalone Python processes on each device.

**Tech Stack:** Python 3.12+, FastAPI, uvicorn, WebSockets, SQLAlchemy 2.0 (async), Alembic, TimescaleDB, Redis (via redis-py), Scapy, MessagePack (msgpack), Pydantic v2, Typer, Docker Compose.

---

## File Structure

```
netsentinel/
├── pyproject.toml                      # Root Python project config
├── docker-compose.yml                  # TimescaleDB, Redis, Core
├── Dockerfile.core                     # Core server container
├── .env.example                        # Template environment variables
├── .gitignore                          # Updated with Python/Docker ignores
│
├── core/                               # FastAPI Core server
│   ���── __init__.py
│   ├── main.py                         # FastAPI app factory + lifespan
│   ├── config.py                       # Settings via pydantic-settings
│   ├��─ database.py                     # Async SQLAlchemy engine + session
│   ├��─ redis_client.py                 # Redis connection + device state helpers
│   ├── models/
│   │   ├── __init__.py
│   │   ├── device.py                   # Device SQLAlchemy model
│   │   └── packet.py                   # Packet SQLAlchemy model
│   ├── schemas/
│   │   ├─��� __init__.py
│   │   ├── device.py                   # Device Pydantic schemas
│   │   ├── packet.py                   # Packet Pydantic schemas
│   │   └── agent_protocol.py          # MessagePack message schemas
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py                     # Dependency injection (db session, redis)
│   │   ├── health.py                   # GET /api/health
│   │   ├── devices.py                  # GET /api/devices, GET /api/devices/{id}
│   │   └── packets.py                  # GET /api/devices/{id}/packets
│   ├── services/
│   │   ├── __init__.py
│   │   ├── agent_manager.py            # WebSocket server, agent registry
│   │   └── packet_engine.py            # L1: parse + store packets
│   └── migrations/
│       ├── env.py                      # Alembic async env
│       ├── script.py.mako
│       └── versions/
│           └── 001_initial_schema.py   # devices + packets tables
│
├── agent/
│   ├── __init__.py
│   ├── main.py                         # Typer CLI: setup/start/stop/status
│   ├── config.py                       # Agent config (YAML file)
│   ├── transport.py                    # WebSocket client + MessagePack
│   └── capture/
│       ├── __init__.py
│       └── scapy_capture.py            # Scapy packet capture module
│
└── tests/
    ├── conftest.py                     # Fixtures: async client, test db, redis mock
    ├── core/
    │   ├── test_health.py
    │   ├��─ test_config.py
    │   ├── test_database.py
    │   ├── test_redis_client.py
    │   ├── test_models.py
    │   ├── test_agent_manager.py
    │   ├── test_packet_engine.py
    │   ├── test_devices_api.py
    │   └── test_packets_api.py
    └── agent/
        ├── test_config.py
        ├── test_transport.py
        └── test_scapy_capture.py
```

---

## Task 1: Project Skeleton + Dependencies

**Files:**
- Create: `pyproject.toml`
- Create: `.env.example`
- Modify: `.gitignore`
- Create: `core/__init__.py`
- Create: `core/config.py`
- Create: `tests/conftest.py`
- Create: `tests/core/test_config.py`

- [ ] **Step 1: Create `pyproject.toml` with all Phase 1 dependencies**

```toml
[project]
name = "netsentinel"
version = "0.1.0"
description = "Personal Network Operations Center"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
    "pydantic>=2.7.0",
    "pydantic-settings>=2.3.0",
    "redis>=5.0.0",
    "msgpack>=1.0.8",
    "websockets>=12.0",
    "scapy>=2.5.0",
    "typer>=0.12.0",
    "rich>=13.7.0",
    "pyyaml>=6.0.1",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.2.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.0",
    "fakeredis>=2.23.0",
    "ruff>=0.4.0",
]

[project.scripts]
netsen-agent = "agent.main:app"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
target-version = "py312"
line-length = 100
```

- [ ] **Step 2: Create `.env.example`**

```bash
# Database
DATABASE_URL=postgresql+asyncpg://netsentinel:netsentinel_dev@localhost:5432/netsentinel

# Redis
REDIS_URL=redis://localhost:6379/0

# Auth
JWT_SECRET=change_me_to_random_64_char_string
ADMIN_USERNAME=admin
ADMIN_PASSWORD=change_me

# Agent
AGENT_SECRET_KEY=change_me_to_random_agent_key
```

- [ ] **Step 3: Update `.gitignore`**

```gitignore
# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
.venv/
venv/

# Env
.env
.env.*
!.env.example

# Docker
*.log

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# NetSentinel
forensics/
ml_models/*.pkl
agent.conf
```

- [ ] **Step 4: Create `core/__init__.py` and `core/config.py`**

`core/__init__.py`:
```python
```

`core/config.py`:
```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://netsentinel:netsentinel_dev@localhost:5432/netsentinel"
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str = "change_me"
    admin_username: str = "admin"
    admin_password: str = "change_me"
    agent_secret_key: str = "change_me"

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}


settings = Settings()
```

- [ ] **Step 5: Write test for config loading**

`tests/__init__.py`: (empty)
`tests/core/__init__.py`: (empty)

`tests/core/test_config.py`:
```python
import os

from core.config import Settings


def test_settings_defaults():
    s = Settings(
        _env_file=None,
        database_url="postgresql+asyncpg://test:test@localhost/test",
        redis_url="redis://localhost:6379/0",
        jwt_secret="test_secret",
        admin_username="admin",
        admin_password="pass",
        agent_secret_key="agent_key",
    )
    assert s.database_url == "postgresql+asyncpg://test:test@localhost/test"
    assert s.jwt_secret == "test_secret"


def test_settings_from_env(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://env:env@db/envdb")
    monkeypatch.setenv("JWT_SECRET", "env_secret")
    s = Settings(_env_file=None)
    assert s.database_url == "postgresql+asyncpg://env:env@db/envdb"
    assert s.jwt_secret == "env_secret"
```

- [ ] **Step 6: Create minimal `tests/conftest.py`**

```python
import pytest
```

- [ ] **Step 7: Install dependencies and run test**

```bash
cd /Users/ayush18/netsentinel
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
pytest tests/core/test_config.py -v
```

Expected: 2 tests PASS.

- [ ] **Step 8: Commit**

```bash
git add pyproject.toml .env.example .gitignore core/__init__.py core/config.py tests/
git commit -m "feat: project skeleton with dependencies and config"
```

---

## Task 2: Docker Compose (TimescaleDB + Redis)

**Files:**
- Create: `docker-compose.yml`
- Create: `Dockerfile.core`
- Create: `scripts/init-db.sql`

- [ ] **Step 1: Create `docker-compose.yml`**

```yaml
services:
  timescaledb:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_USER: netsentinel
      POSTGRES_PASSWORD: netsentinel_dev
      POSTGRES_DB: netsentinel
    ports:
      - "5432:5432"
    volumes:
      - timescaledb_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U netsentinel"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  timescaledb_data:
  redis_data:
```

- [ ] **Step 2: Create `scripts/init-db.sql`**

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

- [ ] **Step 3: Create `Dockerfile.core`** (for later use — not in compose yet)

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY pyproject.toml .
RUN pip install --no-cache-dir .

COPY core/ core/

CMD ["uvicorn", "core.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] **Step 4: Start services and verify**

```bash
docker-compose up -d
docker-compose ps
```

Expected: Both `timescaledb` and `redis` show as `healthy`.

```bash
# Verify TimescaleDB
docker exec netsentinel-timescaledb-1 psql -U netsentinel -c "SELECT extname FROM pg_extension WHERE extname = 'timescaledb';"

# Verify Redis
docker exec netsentinel-redis-1 redis-cli ping
```

Expected: TimescaleDB extension exists, Redis returns `PONG`.

- [ ] **Step 5: Commit**

```bash
git add docker-compose.yml Dockerfile.core scripts/
git commit -m "feat: Docker Compose with TimescaleDB and Redis"
```

---

## Task 3: Database Layer (SQLAlchemy + Alembic)

**Files:**
- Create: `core/database.py`
- Create: `core/models/__init__.py`
- Create: `core/models/device.py`
- Create: `core/models/packet.py`
- Create: `core/migrations/env.py`
- Create: `core/migrations/script.py.mako`
- Create: `core/migrations/versions/001_initial_schema.py`
- Create: `alembic.ini`
- Create: `tests/core/test_database.py`
- Create: `tests/core/test_models.py`

- [ ] **Step 1: Create `core/database.py`**

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from core.config import settings

engine = create_async_engine(settings.database_url, echo=False)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_session() -> AsyncSession:
    async with async_session() as session:
        yield session
```

- [ ] **Step 2: Create `core/models/__init__.py`**

```python
from core.models.device import Device
from core.models.packet import Packet

__all__ = ["Device", "Packet"]
```

- [ ] **Step 3: Create `core/models/device.py`**

```python
from datetime import datetime

from sqlalchemy import String, Text, DateTime, func
from sqlalchemy.dialects.postgresql import INET, JSONB
from sqlalchemy.orm import Mapped, mapped_column

from core.database import Base


class Device(Base):
    __tablename__ = "devices"

    id: Mapped[str] = mapped_column(String(64), primary_key=True)
    name: Mapped[str] = mapped_column(String(128), nullable=False)
    device_type: Mapped[str] = mapped_column(String(32), nullable=False)
    last_seen: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    network_type: Mapped[str | None] = mapped_column(String(16))
    local_ip: Mapped[str | None] = mapped_column(INET)
    public_ip: Mapped[str | None] = mapped_column(INET)
    status: Mapped[str] = mapped_column(String(16), default="offline")
    agent_version: Mapped[str | None] = mapped_column(String(32))
    os_info: Mapped[str | None] = mapped_column(String(128))
    metadata_: Mapped[dict] = mapped_column("metadata", JSONB, default=dict)
    registered_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
```

- [ ] **Step 4: Create `core/models/packet.py`**

```python
from datetime import datetime
from uuid import UUID

from sqlalchemy import BigInteger, Integer, String, Text, DateTime, func
from sqlalchemy.dialects.postgresql import INET, JSONB, UUID as PGUUID
from sqlalchemy.orm import Mapped, mapped_column

from core.database import Base


class Packet(Base):
    __tablename__ = "packets"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    device_id: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    timestamp: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    src_ip: Mapped[str] = mapped_column(INET, nullable=False)
    dst_ip: Mapped[str] = mapped_column(INET, nullable=False)
    src_port: Mapped[int | None] = mapped_column(Integer)
    dst_port: Mapped[int | None] = mapped_column(Integer)
    protocol: Mapped[str] = mapped_column(String(16), nullable=False)
    payload_size: Mapped[int] = mapped_column(Integer, default=0)
    flags: Mapped[str | None] = mapped_column(String(32))
    direction: Mapped[str | None] = mapped_column(String(16))
    flow_id: Mapped[UUID | None] = mapped_column(PGUUID(as_uuid=True))
    raw_summary: Mapped[dict | None] = mapped_column(JSONB)
```

- [ ] **Step 5: Create `alembic.ini`**

```ini
[alembic]
script_location = core/migrations
sqlalchemy.url = postgresql+asyncpg://netsentinel:netsentinel_dev@localhost:5432/netsentinel

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
```

- [ ] **Step 6: Create `core/migrations/env.py`**

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine

from core.config import settings
from core.database import Base
from core.models import Device, Packet  # noqa: F401 - register models

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    context.configure(
        url=settings.database_url,
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


async def run_migrations_online() -> None:
    connectable = create_async_engine(settings.database_url)
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

- [ ] **Step 7: Create `core/migrations/script.py.mako`**

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

revision: str = ${repr(up_revision)}
down_revision: Union[str, None] = ${repr(down_revision)}
branch_labels: Union[str, Sequence[str], None] = ${repr(branch_labels)}
depends_on: Union[str, Sequence[str], None] = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

- [ ] **Step 8: Create `core/migrations/versions/001_initial_schema.py`**

```python
"""Initial schema: devices and packets tables

Revision ID: 001
Revises: None
Create Date: 2026-04-09
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import INET, JSONB, UUID

revision: str = "001"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # Devices table
    op.create_table(
        "devices",
        sa.Column("id", sa.String(64), primary_key=True),
        sa.Column("name", sa.String(128), nullable=False),
        sa.Column("device_type", sa.String(32), nullable=False),
        sa.Column("last_seen", sa.DateTime(timezone=True)),
        sa.Column("network_type", sa.String(16)),
        sa.Column("local_ip", INET),
        sa.Column("public_ip", INET),
        sa.Column("status", sa.String(16), server_default="offline"),
        sa.Column("agent_version", sa.String(32)),
        sa.Column("os_info", sa.String(128)),
        sa.Column("metadata", JSONB, server_default="{}"),
        sa.Column("registered_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )
    op.create_index("idx_devices_status", "devices", ["status"])

    # Packets table
    op.create_table(
        "packets",
        sa.Column("id", sa.BigInteger, primary_key=True, autoincrement=True),
        sa.Column("device_id", sa.String(64), nullable=False),
        sa.Column("timestamp", sa.DateTime(timezone=True), nullable=False),
        sa.Column("src_ip", INET, nullable=False),
        sa.Column("dst_ip", INET, nullable=False),
        sa.Column("src_port", sa.Integer),
        sa.Column("dst_port", sa.Integer),
        sa.Column("protocol", sa.String(16), nullable=False),
        sa.Column("payload_size", sa.Integer, server_default="0"),
        sa.Column("flags", sa.String(32)),
        sa.Column("direction", sa.String(16)),
        sa.Column("flow_id", UUID(as_uuid=True)),
        sa.Column("raw_summary", JSONB),
    )
    op.create_index("idx_packets_device", "packets", ["device_id", sa.text("timestamp DESC")])
    op.create_index("idx_packets_protocol", "packets", ["protocol"])

    # Convert packets to hypertable
    op.execute("SELECT create_hypertable('packets', 'timestamp', chunk_time_interval => INTERVAL '1 hour', migrate_data => true)")


def downgrade() -> None:
    op.drop_table("packets")
    op.drop_table("devices")
```

- [ ] **Step 9: Write model test**

`tests/core/test_models.py`:
```python
from core.models.device import Device
from core.models.packet import Packet


def test_device_model_has_correct_tablename():
    assert Device.__tablename__ == "devices"


def test_device_model_columns():
    columns = {c.name for c in Device.__table__.columns}
    assert "id" in columns
    assert "name" in columns
    assert "device_type" in columns
    assert "status" in columns
    assert "metadata" in columns


def test_packet_model_has_correct_tablename():
    assert Packet.__tablename__ == "packets"


def test_packet_model_columns():
    columns = {c.name for c in Packet.__table__.columns}
    assert "id" in columns
    assert "device_id" in columns
    assert "timestamp" in columns
    assert "src_ip" in columns
    assert "dst_ip" in columns
    assert "protocol" in columns
    assert "payload_size" in columns
```

- [ ] **Step 10: Run test**

```bash
pytest tests/core/test_models.py -v
```

Expected: 4 tests PASS.

- [ ] **Step 11: Run Alembic migration against live DB**

```bash
# Ensure Docker services are running
docker-compose up -d

# Run migration
alembic upgrade head
```

Expected: Migration `001` applied. Verify:

```bash
docker exec netsentinel-timescaledb-1 psql -U netsentinel -c "\dt"
```

Expected: `devices` and `packets` tables exist.

```bash
docker exec netsentinel-timescaledb-1 psql -U netsentinel -c "SELECT * FROM timescaledb_information.hypertables WHERE hypertable_name = 'packets';"
```

Expected: `packets` shows as hypertable.

- [ ] **Step 12: Commit**

```bash
git add core/database.py core/models/ alembic.ini core/migrations/ tests/core/test_models.py
git commit -m "feat: database layer with Device + Packet models and Alembic migration"
```

---

## Task 4: Redis Client + Device State Helpers

**Files:**
- Create: `core/redis_client.py`
- Create: `tests/core/test_redis_client.py`

- [ ] **Step 1: Write failing tests for Redis device state helpers**

`tests/core/test_redis_client.py`:
```python
import pytest
import fakeredis.aioredis

from core.redis_client import DeviceStateManager


@pytest.fixture
async def redis():
    r = fakeredis.aioredis.FakeRedis()
    yield r
    await r.aclose()


@pytest.fixture
def state_manager(redis):
    return DeviceStateManager(redis)


async def test_set_device_online(state_manager, redis):
    await state_manager.set_online("phone-1", network_type="wifi", local_ip="192.168.1.5")
    status = await redis.hget("device:phone-1:status", "status")
    assert status == b"online"


async def test_set_device_offline(state_manager, redis):
    await state_manager.set_online("phone-1", network_type="wifi", local_ip="192.168.1.5")
    await state_manager.set_offline("phone-1")
    status = await redis.hget("device:phone-1:status", "status")
    assert status == b"offline"


async def test_get_device_status(state_manager):
    await state_manager.set_online("laptop", network_type="ethernet", local_ip="192.168.1.10")
    info = await state_manager.get_status("laptop")
    assert info["status"] == "online"
    assert info["network_type"] == "ethernet"


async def test_get_status_nonexistent(state_manager):
    info = await state_manager.get_status("ghost-device")
    assert info is None


async def test_get_all_online_devices(state_manager):
    await state_manager.set_online("phone-1", network_type="wifi", local_ip="10.0.0.1")
    await state_manager.set_online("laptop", network_type="wifi", local_ip="10.0.0.2")
    await state_manager.set_offline("phone-1")
    online = await state_manager.get_online_device_ids()
    assert "laptop" in online
    assert "phone-1" not in online


async def test_update_bandwidth(state_manager, redis):
    await state_manager.update_bandwidth("phone-1", bytes_in=1024, bytes_out=2048)
    bw = await redis.hgetall("device:phone-1:bandwidth")
    assert bw[b"bytes_in_sec"] == b"1024"
    assert bw[b"bytes_out_sec"] == b"2048"
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest tests/core/test_redis_client.py -v
```

Expected: FAIL — `core.redis_client` does not exist.

- [ ] **Step 3: Implement `core/redis_client.py`**

```python
from datetime import datetime, timezone

from redis.asyncio import Redis

from core.config import settings


async def get_redis() -> Redis:
    return Redis.from_url(settings.redis_url, decode_responses=False)


class DeviceStateManager:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def set_online(self, device_id: str, network_type: str, local_ip: str) -> None:
        key = f"device:{device_id}:status"
        now = datetime.now(timezone.utc).isoformat()
        await self.redis.hset(key, mapping={
            "status": "online",
            "last_heartbeat": now,
            "network_type": network_type,
            "local_ip": local_ip,
        })
        await self.redis.expire(key, 120)
        await self.redis.sadd("devices:online", device_id)

    async def set_offline(self, device_id: str) -> None:
        key = f"device:{device_id}:status"
        await self.redis.hset(key, "status", "offline")
        await self.redis.persist(key)
        await self.redis.srem("devices:online", device_id)

    async def get_status(self, device_id: str) -> dict | None:
        key = f"device:{device_id}:status"
        data = await self.redis.hgetall(key)
        if not data:
            return None
        return {k.decode(): v.decode() for k, v in data.items()}

    async def get_online_device_ids(self) -> list[str]:
        members = await self.redis.smembers("devices:online")
        return [m.decode() for m in members]

    async def update_bandwidth(self, device_id: str, bytes_in: int, bytes_out: int) -> None:
        key = f"device:{device_id}:bandwidth"
        await self.redis.hset(key, mapping={
            "bytes_in_sec": str(bytes_in),
            "bytes_out_sec": str(bytes_out),
        })
        await self.redis.expire(key, 10)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
pytest tests/core/test_redis_client.py -v
```

Expected: 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add core/redis_client.py tests/core/test_redis_client.py
git commit -m "feat: Redis device state manager with online/offline/bandwidth tracking"
```

---

## Task 5: FastAPI App + Health Endpoint

**Files:**
- Create: `core/main.py`
- Create: `core/api/__init__.py`
- Create: `core/api/deps.py`
- Create: `core/api/health.py`
- Create: `core/schemas/__init__.py`
- Create: `tests/core/test_health.py`

- [ ] **Step 1: Write failing test for health endpoint**

`tests/core/test_health.py`:
```python
import pytest
from httpx import ASGITransport, AsyncClient

from core.main import app


@pytest.fixture
async def client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c


async def test_health_returns_200(client):
    response = await client.get("/api/health")
    assert response.status_code == 200


async def test_health_has_status(client):
    response = await client.get("/api/health")
    data = response.json()
    assert data["status"] == "healthy"
    assert "version" in data
    assert "uptime_sec" in data
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pytest tests/core/test_health.py -v
```

Expected: FAIL — `core.main` does not exist.

- [ ] **Step 3: Create `core/schemas/__init__.py`**

```python
```

- [ ] **Step 4: Create `core/api/__init__.py`**

```python
```

- [ ] **Step 5: Create `core/api/deps.py`**

```python
from sqlalchemy.ext.asyncio import AsyncSession
from redis.asyncio import Redis

from core.database import async_session
from core.redis_client import get_redis


async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session


async def get_redis_client() -> Redis:
    redis = await get_redis()
    try:
        yield redis
    finally:
        await redis.aclose()
```

- [ ] **Step 6: Create `core/api/health.py`**

```python
import time

from fastapi import APIRouter

router = APIRouter()

_start_time = time.time()


@router.get("/health")
async def health():
    return {
        "status": "healthy",
        "version": "0.1.0",
        "uptime_sec": round(time.time() - _start_time),
    }
```

- [ ] **Step 7: Create `core/main.py`**

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from core.api.health import router as health_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield


app = FastAPI(title="NetSentinel", version="0.1.0", lifespan=lifespan)
app.include_router(health_router, prefix="/api")
```

- [ ] **Step 8: Run tests**

```bash
pytest tests/core/test_health.py -v
```

Expected: 2 tests PASS.

- [ ] **Step 9: Manual verification**

```bash
uvicorn core.main:app --reload --port 8000
# In another terminal:
curl http://localhost:8000/api/health
```

Expected: `{"status":"healthy","version":"0.1.0","uptime_sec":0}`

- [ ] **Step 10: Commit**

```bash
git add core/main.py core/api/ core/schemas/__init__.py tests/core/test_health.py
git commit -m "feat: FastAPI app with health endpoint"
```

---

## Task 6: Pydantic Schemas + Device/Packet API Endpoints

**Files:**
- Create: `core/schemas/device.py`
- Create: `core/schemas/packet.py`
- Create: `core/api/devices.py`
- Create: `core/api/packets.py`
- Create: `tests/core/test_devices_api.py`
- Create: `tests/core/test_packets_api.py`

- [ ] **Step 1: Create `core/schemas/device.py`**

```python
from datetime import datetime

from pydantic import BaseModel


class DeviceResponse(BaseModel):
    id: str
    name: str
    device_type: str
    status: str
    last_seen: datetime | None = None
    network_type: str | None = None
    local_ip: str | None = None
    public_ip: str | None = None
    agent_version: str | None = None
    os_info: str | None = None
    registered_at: datetime | None = None

    model_config = {"from_attributes": True}


class DeviceListResponse(BaseModel):
    devices: list[DeviceResponse]
    total: int
    online: int
```

- [ ] **Step 2: Create `core/schemas/packet.py`**

```python
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel


class PacketResponse(BaseModel):
    id: int
    device_id: str
    timestamp: datetime
    src_ip: str
    dst_ip: str
    src_port: int | None = None
    dst_port: int | None = None
    protocol: str
    payload_size: int
    flags: str | None = None
    direction: str | None = None
    flow_id: UUID | None = None
    raw_summary: dict | None = None

    model_config = {"from_attributes": True}


class PacketListResponse(BaseModel):
    packets: list[PacketResponse]
    total: int
```

- [ ] **Step 3: Create `core/api/devices.py`**

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession

from core.api.deps import get_db
from core.models.device import Device
from core.schemas.device import DeviceResponse, DeviceListResponse

router = APIRouter()


@router.get("/devices", response_model=DeviceListResponse)
async def list_devices(
    status: str | None = None,
    db: AsyncSession = Depends(get_db),
):
    query = select(Device)
    if status:
        query = query.where(Device.status == status)

    result = await db.execute(query)
    devices = result.scalars().all()

    online_count = sum(1 for d in devices if d.status == "online")

    return DeviceListResponse(
        devices=[DeviceResponse.model_validate(d) for d in devices],
        total=len(devices),
        online=online_count,
    )


@router.get("/devices/{device_id}", response_model=DeviceResponse)
async def get_device(device_id: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Device).where(Device.id == device_id))
    device = result.scalar_one_or_none()
    if not device:
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")
    return DeviceResponse.model_validate(device)
```

- [ ] **Step 4: Create `core/api/packets.py`**

```python
from datetime import datetime, timezone, timedelta

from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy import select, func, desc
from sqlalchemy.ext.asyncio import AsyncSession

from core.api.deps import get_db
from core.models.device import Device
from core.models.packet import Packet
from core.schemas.packet import PacketResponse, PacketListResponse

router = APIRouter()


@router.get("/devices/{device_id}/packets", response_model=PacketListResponse)
async def list_packets(
    device_id: str,
    protocol: str | None = None,
    src_ip: str | None = None,
    dst_ip: str | None = None,
    since_minutes: int = Query(default=5, ge=1, le=1440),
    limit: int = Query(default=100, ge=1, le=1000),
    db: AsyncSession = Depends(get_db),
):
    # Verify device exists
    device_result = await db.execute(select(Device).where(Device.id == device_id))
    if not device_result.scalar_one_or_none():
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")

    since = datetime.now(timezone.utc) - timedelta(minutes=since_minutes)

    query = (
        select(Packet)
        .where(Packet.device_id == device_id)
        .where(Packet.timestamp >= since)
    )

    if protocol:
        query = query.where(Packet.protocol == protocol)
    if src_ip:
        query = query.where(Packet.src_ip == src_ip)
    if dst_ip:
        query = query.where(Packet.dst_ip == dst_ip)

    # Count total
    count_query = select(func.count()).select_from(query.subquery())
    total_result = await db.execute(count_query)
    total = total_result.scalar()

    # Fetch with limit
    query = query.order_by(desc(Packet.timestamp)).limit(limit)
    result = await db.execute(query)
    packets = result.scalars().all()

    return PacketListResponse(
        packets=[PacketResponse.model_validate(p) for p in packets],
        total=total,
    )
```

- [ ] **Step 5: Register routers in `core/main.py`**

Replace `core/main.py`:
```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from core.api.health import router as health_router
from core.api.devices import router as devices_router
from core.api.packets import router as packets_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield


app = FastAPI(title="NetSentinel", version="0.1.0", lifespan=lifespan)
app.include_router(health_router, prefix="/api")
app.include_router(devices_router, prefix="/api")
app.include_router(packets_router, prefix="/api")
```

- [ ] **Step 6: Write device API tests**

`tests/core/test_devices_api.py`:
```python
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from core.database import Base
from core.main import app
from core.api.deps import get_db
from core.models.device import Device


# Use SQLite for tests (no TimescaleDB needed for basic CRUD)
TEST_DB_URL = "sqlite+aiosqlite:///test_devices.db"


@pytest.fixture(autouse=True)
async def setup_db():
    engine = create_async_engine(TEST_DB_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

    async def override_get_db():
        async with session_factory() as session:
            yield session

    app.dependency_overrides[get_db] = override_get_db

    yield session_factory

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

    import os
    if os.path.exists("test_devices.db"):
        os.remove("test_devices.db")

    app.dependency_overrides.clear()


@pytest.fixture
async def client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c


@pytest.fixture
async def seeded_db(setup_db):
    async with setup_db() as session:
        session.add(Device(
            id="phone-1", name="Test Phone", device_type="android", status="online"
        ))
        session.add(Device(
            id="laptop", name="Test Laptop", device_type="laptop", status="offline"
        ))
        await session.commit()


async def test_list_devices_empty(client):
    resp = await client.get("/api/devices")
    assert resp.status_code == 200
    data = resp.json()
    assert data["total"] == 0
    assert data["devices"] == []


async def test_list_devices(client, seeded_db):
    resp = await client.get("/api/devices")
    assert resp.status_code == 200
    data = resp.json()
    assert data["total"] == 2
    assert data["online"] == 1


async def test_list_devices_filter_status(client, seeded_db):
    resp = await client.get("/api/devices?status=online")
    data = resp.json()
    assert data["total"] == 1
    assert data["devices"][0]["id"] == "phone-1"


async def test_get_device(client, seeded_db):
    resp = await client.get("/api/devices/phone-1")
    assert resp.status_code == 200
    assert resp.json()["name"] == "Test Phone"


async def test_get_device_not_found(client):
    resp = await client.get("/api/devices/nonexistent")
    assert resp.status_code == 404
```

- [ ] **Step 7: Run tests**

```bash
pip install aiosqlite  # Needed for test DB
pytest tests/core/test_devices_api.py -v
```

Expected: 5 tests PASS.

- [ ] **Step 8: Commit**

```bash
git add core/schemas/ core/api/ core/main.py tests/core/test_devices_api.py
git commit -m "feat: Device + Packet REST API endpoints with schemas"
```

---

## Task 7: Agent Protocol Schemas

**Files:**
- Create: `core/schemas/agent_protocol.py`

- [ ] **Step 1: Create `core/schemas/agent_protocol.py`**

```python
from datetime import datetime
from enum import Enum

from pydantic import BaseModel


class MessageType(str, Enum):
    HEARTBEAT = "HEARTBEAT"
    FLOW_REPORT = "FLOW_REPORT"
    PACKET_FLAG = "PACKET_FLAG"
    BULK_STATS = "BULK_STATS"
    COMMAND = "COMMAND"
    ACK = "ACK"


class AgentMessage(BaseModel):
    type: MessageType
    device_id: str
    timestamp: float
    seq: int
    payload: dict


class HeartbeatPayload(BaseModel):
    network_type: str
    local_ip: str
    battery: int | None = None
    cpu_percent: float | None = None
    agent_version: str = "0.1.0"
    os_info: str | None = None


class PacketPayload(BaseModel):
    src_ip: str
    dst_ip: str
    src_port: int | None = None
    dst_port: int | None = None
    protocol: str
    payload_size: int = 0
    flags: str | None = None
    direction: str | None = None
    raw_summary: dict | None = None


class CommandPayload(BaseModel):
    action: str  # "block_ip", "unblock_ip", "quarantine", "unquarantine"
    target: str | None = None  # IP or domain
    reason: str | None = None
```

- [ ] **Step 2: Commit**

```bash
git add core/schemas/agent_protocol.py
git commit -m "feat: agent protocol message schemas (MessagePack format)"
```

---

## Task 8: WebSocket Agent Manager (Core Side)

**Files:**
- Create: `core/services/__init__.py`
- Create: `core/services/agent_manager.py`
- Create: `tests/core/test_agent_manager.py`

- [ ] **Step 1: Write failing test for agent manager**

`tests/core/test_agent_manager.py`:
```python
import pytest
import msgpack
import fakeredis.aioredis

from core.services.agent_manager import AgentManager
from core.schemas.agent_protocol import MessageType


@pytest.fixture
async def redis():
    r = fakeredis.aioredis.FakeRedis()
    yield r
    await r.aclose()


@pytest.fixture
def manager(redis):
    return AgentManager(redis=redis, agent_secret_key="test_secret")


def test_verify_key_valid(manager):
    assert manager.verify_key("test_secret") is True


def test_verify_key_invalid(manager):
    assert manager.verify_key("wrong_key") is False


def test_parse_message_valid(manager):
    raw = msgpack.packb({
        "type": "HEARTBEAT",
        "device_id": "phone-1",
        "timestamp": 1712678400.0,
        "seq": 1,
        "payload": {"network_type": "wifi", "local_ip": "192.168.1.5"},
    })
    msg = manager.parse_message(raw)
    assert msg.type == MessageType.HEARTBEAT
    assert msg.device_id == "phone-1"


def test_parse_message_invalid(manager):
    msg = manager.parse_message(b"not valid msgpack at all \x00\x01")
    assert msg is None


async def test_handle_heartbeat(manager):
    await manager.handle_heartbeat("phone-1", {
        "network_type": "wifi",
        "local_ip": "192.168.1.5",
        "agent_version": "0.1.0",
    })
    status = await manager.redis.hget("device:phone-1:status", "status")
    assert status == b"online"
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest tests/core/test_agent_manager.py -v
```

Expected: FAIL — `core.services.agent_manager` does not exist.

- [ ] **Step 3: Create `core/services/__init__.py`**

```python
```

- [ ] **Step 4: Create `core/services/agent_manager.py`**

```python
import logging
import time

import msgpack
from redis.asyncio import Redis
from fastapi import WebSocket

from core.schemas.agent_protocol import AgentMessage, MessageType
from core.redis_client import DeviceStateManager

logger = logging.getLogger(__name__)


class AgentManager:
    def __init__(self, redis: Redis, agent_secret_key: str):
        self.redis = redis
        self.agent_secret_key = agent_secret_key
        self.state = DeviceStateManager(redis)
        self.connections: dict[str, WebSocket] = {}

    def verify_key(self, key: str) -> bool:
        return key == self.agent_secret_key

    def parse_message(self, raw: bytes) -> AgentMessage | None:
        try:
            data = msgpack.unpackb(raw, raw=False)
            return AgentMessage(**data)
        except Exception:
            logger.warning("Failed to parse agent message")
            return None

    async def handle_heartbeat(self, device_id: str, payload: dict) -> None:
        network_type = payload.get("network_type", "unknown")
        local_ip = payload.get("local_ip", "0.0.0.0")
        await self.state.set_online(device_id, network_type=network_type, local_ip=local_ip)

    async def register_connection(self, device_id: str, ws: WebSocket) -> None:
        self.connections[device_id] = ws
        logger.info(f"Agent connected: {device_id}")

    async def unregister_connection(self, device_id: str) -> None:
        self.connections.pop(device_id, None)
        await self.state.set_offline(device_id)
        logger.info(f"Agent disconnected: {device_id}")

    async def send_command(self, device_id: str, action: str, target: str | None = None) -> bool:
        ws = self.connections.get(device_id)
        if not ws:
            return False
        msg = msgpack.packb({
            "type": "COMMAND",
            "device_id": device_id,
            "timestamp": time.time(),
            "seq": 0,
            "payload": {"action": action, "target": target},
        })
        await ws.send_bytes(msg)
        return True
```

- [ ] **Step 5: Run tests**

```bash
pytest tests/core/test_agent_manager.py -v
```

Expected: 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add core/services/ tests/core/test_agent_manager.py
git commit -m "feat: WebSocket agent manager with MessagePack parsing and heartbeat handling"
```

---

## Task 9: WebSocket Endpoint + Agent Connection Flow

**Files:**
- Modify: `core/main.py`
- Create: `core/api/agent_ws.py`

- [ ] **Step 1: Create `core/api/agent_ws.py`**

```python
import logging

from fastapi import APIRouter, WebSocket, WebSocketDisconnect
from redis.asyncio import Redis

from core.config import settings
from core.redis_client import get_redis
from core.services.agent_manager import AgentManager
from core.schemas.agent_protocol import MessageType

logger = logging.getLogger(__name__)

router = APIRouter()


@router.websocket("/ws/agent")
async def agent_websocket(websocket: WebSocket):
    # Check API key from query param or header
    key = websocket.query_params.get("key", "")
    if not key:
        key = websocket.headers.get("x-agent-key", "")

    redis = await get_redis()
    manager = AgentManager(redis=redis, agent_secret_key=settings.agent_secret_key)

    if not manager.verify_key(key):
        await websocket.close(code=4001, reason="Unauthorized")
        await redis.aclose()
        return

    await websocket.accept()

    # Get device_id from first message (heartbeat)
    device_id = None
    try:
        while True:
            raw = await websocket.receive_bytes()
            msg = manager.parse_message(raw)
            if msg is None:
                continue

            if device_id is None:
                device_id = msg.device_id
                await manager.register_connection(device_id, websocket)

            if msg.type == MessageType.HEARTBEAT:
                await manager.handle_heartbeat(msg.device_id, msg.payload)

            elif msg.type == MessageType.PACKET_FLAG:
                # Will be handled by packet engine in Task 11
                logger.debug(f"Packet from {msg.device_id}: {msg.payload.get('src_ip')} -> {msg.payload.get('dst_ip')}")

            elif msg.type == MessageType.BULK_STATS:
                logger.debug(f"Bulk stats from {msg.device_id}")

    except WebSocketDisconnect:
        if device_id:
            await manager.unregister_connection(device_id)
    finally:
        await redis.aclose()
```

- [ ] **Step 2: Register WebSocket router in `core/main.py`**

Replace `core/main.py`:
```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from core.api.health import router as health_router
from core.api.devices import router as devices_router
from core.api.packets import router as packets_router
from core.api.agent_ws import router as agent_ws_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield


app = FastAPI(title="NetSentinel", version="0.1.0", lifespan=lifespan)
app.include_router(health_router, prefix="/api")
app.include_router(devices_router, prefix="/api")
app.include_router(packets_router, prefix="/api")
app.include_router(agent_ws_router)
```

- [ ] **Step 3: Commit**

```bash
git add core/api/agent_ws.py core/main.py
git commit -m "feat: WebSocket endpoint for agent connections with auth and heartbeat"
```

---

## Task 10: Agent Skeleton (CLI + Config + Transport)

**Files:**
- Create: `agent/__init__.py`
- Create: `agent/config.py`
- Create: `agent/transport.py`
- Create: `agent/main.py`
- Create: `tests/agent/__init__.py`
- Create: `tests/agent/test_config.py`
- Create: `tests/agent/test_transport.py`

- [ ] **Step 1: Write test for agent config**

`tests/agent/__init__.py`: (empty)

`tests/agent/test_config.py`:
```python
import os
import tempfile

import yaml

from agent.config import AgentConfig


def test_default_config():
    cfg = AgentConfig()
    assert cfg.device_name == "unnamed-device"
    assert cfg.heartbeat_interval == 30
    assert cfg.capture_mode == "scapy"


def test_save_and_load():
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml", delete=False) as f:
        path = f.name

    try:
        cfg = AgentConfig(
            server_url="wss://192.168.1.100:8000/ws/agent",
            device_name="test-phone",
            device_id="phone-test-123",
            api_key="ns_ak_test123",
            capture_mode="tcpdump",
        )
        cfg.save(path)

        loaded = AgentConfig.load(path)
        assert loaded.server_url == "wss://192.168.1.100:8000/ws/agent"
        assert loaded.device_name == "test-phone"
        assert loaded.device_id == "phone-test-123"
        assert loaded.api_key == "ns_ak_test123"
        assert loaded.capture_mode == "tcpdump"
    finally:
        os.unlink(path)
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pytest tests/agent/test_config.py -v
```

Expected: FAIL.

- [ ] **Step 3: Create `agent/__init__.py`**

```python
```

- [ ] **Step 4: Create `agent/config.py`**

```python
import os
import uuid
from pathlib import Path

import yaml
from pydantic import BaseModel


DEFAULT_CONFIG_PATH = os.path.expanduser("~/.netsentinel/agent.conf")


class AgentConfig(BaseModel):
    server_url: str = "ws://localhost:8000/ws/agent"
    device_name: str = "unnamed-device"
    device_id: str = ""
    api_key: str = ""
    capture_mode: str = "scapy"  # "scapy", "tcpdump", "vpn-tun"
    heartbeat_interval: int = 30
    battery_saver: bool = True
    battery_threshold: int = 20
    log_level: str = "info"

    def model_post_init(self, __context) -> None:
        if not self.device_id:
            short_uuid = uuid.uuid4().hex[:8]
            self.device_id = f"{self.device_name}-{short_uuid}"

    def save(self, path: str = DEFAULT_CONFIG_PATH) -> None:
        Path(path).parent.mkdir(parents=True, exist_ok=True)
        with open(path, "w") as f:
            yaml.dump(self.model_dump(), f, default_flow_style=False)

    @classmethod
    def load(cls, path: str = DEFAULT_CONFIG_PATH) -> "AgentConfig":
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)
```

- [ ] **Step 5: Run config tests**

```bash
pytest tests/agent/test_config.py -v
```

Expected: 2 tests PASS.

- [ ] **Step 6: Write transport test**

`tests/agent/test_transport.py`:
```python
import msgpack

from agent.transport import build_heartbeat, build_packet_message


def test_build_heartbeat():
    raw = build_heartbeat(
        device_id="phone-1",
        seq=1,
        network_type="wifi",
        local_ip="192.168.1.5",
    )
    msg = msgpack.unpackb(raw, raw=False)
    assert msg["type"] == "HEARTBEAT"
    assert msg["device_id"] == "phone-1"
    assert msg["seq"] == 1
    assert msg["payload"]["network_type"] == "wifi"


def test_build_packet_message():
    raw = build_packet_message(
        device_id="phone-1",
        seq=5,
        src_ip="192.168.1.5",
        dst_ip="8.8.8.8",
        src_port=54321,
        dst_port=53,
        protocol="UDP",
        payload_size=64,
    )
    msg = msgpack.unpackb(raw, raw=False)
    assert msg["type"] == "PACKET_FLAG"
    assert msg["payload"]["dst_ip"] == "8.8.8.8"
    assert msg["payload"]["protocol"] == "UDP"
```

- [ ] **Step 7: Create `agent/transport.py`**

```python
import asyncio
import logging
import time

import msgpack
import websockets

from agent.config import AgentConfig

logger = logging.getLogger(__name__)


def build_heartbeat(
    device_id: str,
    seq: int,
    network_type: str = "unknown",
    local_ip: str = "0.0.0.0",
    battery: int | None = None,
    cpu_percent: float | None = None,
    agent_version: str = "0.1.0",
    os_info: str | None = None,
) -> bytes:
    return msgpack.packb({
        "type": "HEARTBEAT",
        "device_id": device_id,
        "timestamp": time.time(),
        "seq": seq,
        "payload": {
            "network_type": network_type,
            "local_ip": local_ip,
            "battery": battery,
            "cpu_percent": cpu_percent,
            "agent_version": agent_version,
            "os_info": os_info,
        },
    })


def build_packet_message(
    device_id: str,
    seq: int,
    src_ip: str,
    dst_ip: str,
    src_port: int | None = None,
    dst_port: int | None = None,
    protocol: str = "TCP",
    payload_size: int = 0,
    flags: str | None = None,
    direction: str | None = None,
    raw_summary: dict | None = None,
) -> bytes:
    return msgpack.packb({
        "type": "PACKET_FLAG",
        "device_id": device_id,
        "timestamp": time.time(),
        "seq": seq,
        "payload": {
            "src_ip": src_ip,
            "dst_ip": dst_ip,
            "src_port": src_port,
            "dst_port": dst_port,
            "protocol": protocol,
            "payload_size": payload_size,
            "flags": flags,
            "direction": direction,
            "raw_summary": raw_summary,
        },
    })


class AgentTransport:
    def __init__(self, config: AgentConfig):
        self.config = config
        self.ws = None
        self.seq = 0
        self._running = False

    async def connect(self) -> None:
        url = f"{self.config.server_url}?key={self.config.api_key}"
        self.ws = await websockets.connect(url)
        logger.info(f"Connected to {self.config.server_url}")

    async def disconnect(self) -> None:
        if self.ws:
            await self.ws.close()
            self.ws = None

    def next_seq(self) -> int:
        self.seq += 1
        return self.seq

    async def send_heartbeat(self, network_type: str = "unknown", local_ip: str = "0.0.0.0") -> None:
        if not self.ws:
            return
        msg = build_heartbeat(
            device_id=self.config.device_id,
            seq=self.next_seq(),
            network_type=network_type,
            local_ip=local_ip,
        )
        await self.ws.send(msg)

    async def send_packet(self, **kwargs) -> None:
        if not self.ws:
            return
        msg = build_packet_message(
            device_id=self.config.device_id,
            seq=self.next_seq(),
            **kwargs,
        )
        await self.ws.send(msg)

    async def run_heartbeat_loop(self) -> None:
        self._running = True
        while self._running:
            try:
                await self.send_heartbeat()
                await asyncio.sleep(self.config.heartbeat_interval)
            except Exception as e:
                logger.error(f"Heartbeat error: {e}")
                break

    def stop(self) -> None:
        self._running = False
```

- [ ] **Step 8: Run transport tests**

```bash
pytest tests/agent/test_transport.py -v
```

Expected: 2 tests PASS.

- [ ] **Step 9: Create `agent/main.py`** (Typer CLI)

```python
import asyncio
import logging
import platform
import uuid

import typer
from rich.console import Console
from rich.table import Table

from agent.config import AgentConfig, DEFAULT_CONFIG_PATH
from agent.transport import AgentTransport

app = typer.Typer(name="netsen-agent", help="NetSentinel Device Agent")
console = Console()


@app.command()
def setup(
    server: str = typer.Option(..., help="Core server WebSocket URL"),
    name: str = typer.Option(..., help="Device name"),
    key: str = typer.Option(..., help="Agent API key"),
    capture_mode: str = typer.Option("scapy", help="Capture mode: scapy, tcpdump, vpn-tun"),
):
    """Configure the agent for this device."""
    config = AgentConfig(
        server_url=server,
        device_name=name,
        api_key=key,
        capture_mode=capture_mode,
    )
    config.save()
    console.print(f"[green]Agent configured![/green]")
    console.print(f"  Device ID: {config.device_id}")
    console.print(f"  Server: {config.server_url}")
    console.print(f"  Config saved to: {DEFAULT_CONFIG_PATH}")


@app.command()
def start():
    """Start the agent."""
    try:
        config = AgentConfig.load()
    except FileNotFoundError:
        console.print("[red]No config found. Run 'netsen-agent setup' first.[/red]")
        raise typer.Exit(1)

    console.print(f"[green]Starting NetSentinel Agent[/green]")
    console.print(f"  Device: {config.device_name} ({config.device_id})")
    console.print(f"  Server: {config.server_url}")
    console.print(f"  Capture: {config.capture_mode}")

    asyncio.run(_run_agent(config))


async def _run_agent(config: AgentConfig):
    transport = AgentTransport(config)

    try:
        await transport.connect()
        console.print("[green]Connected to Core![/green]")

        # Send initial heartbeat
        await transport.send_heartbeat(network_type="unknown", local_ip="0.0.0.0")

        # Run heartbeat loop (capture will be added in Task 11)
        await transport.run_heartbeat_loop()

    except KeyboardInterrupt:
        console.print("\n[yellow]Shutting down...[/yellow]")
    except Exception as e:
        console.print(f"[red]Error: {e}[/red]")
    finally:
        transport.stop()
        await transport.disconnect()


@app.command()
def status():
    """Show agent status."""
    try:
        config = AgentConfig.load()
        table = Table(title="Agent Status")
        table.add_column("Field", style="cyan")
        table.add_column("Value", style="green")
        table.add_row("Device Name", config.device_name)
        table.add_row("Device ID", config.device_id)
        table.add_row("Server", config.server_url)
        table.add_row("Capture Mode", config.capture_mode)
        table.add_row("Heartbeat Interval", f"{config.heartbeat_interval}s")
        console.print(table)
    except FileNotFoundError:
        console.print("[red]No config found. Run 'netsen-agent setup' first.[/red]")


if __name__ == "__main__":
    app()
```

- [ ] **Step 10: Commit**

```bash
git add agent/ tests/agent/
git commit -m "feat: agent skeleton with CLI (setup/start/status), config, and WebSocket transport"
```

---

## Task 11: Scapy Packet Capture + L1 Packet Engine

**Files:**
- Create: `agent/capture/__init__.py`
- Create: `agent/capture/scapy_capture.py`
- Create: `core/services/packet_engine.py`
- Create: `tests/agent/test_scapy_capture.py`
- Create: `tests/core/test_packet_engine.py`

- [ ] **Step 1: Write test for packet extraction from Scapy**

`tests/agent/test_scapy_capture.py`:
```python
from scapy.all import IP, TCP, UDP, DNS, DNSQR, Raw

from agent.capture.scapy_capture import extract_packet_info


def test_extract_tcp_packet():
    pkt = IP(src="192.168.1.5", dst="142.250.80.46") / TCP(sport=54321, dport=443, flags="S")
    info = extract_packet_info(pkt)
    assert info["src_ip"] == "192.168.1.5"
    assert info["dst_ip"] == "142.250.80.46"
    assert info["src_port"] == 54321
    assert info["dst_port"] == 443
    assert info["protocol"] == "TCP"
    assert info["flags"] == "S"


def test_extract_udp_packet():
    pkt = IP(src="192.168.1.5", dst="8.8.8.8") / UDP(sport=12345, dport=53)
    info = extract_packet_info(pkt)
    assert info["protocol"] == "UDP"
    assert info["dst_port"] == 53


def test_extract_dns_packet():
    pkt = (
        IP(src="192.168.1.5", dst="8.8.8.8")
        / UDP(sport=12345, dport=53)
        / DNS(rd=1, qd=DNSQR(qname="google.com"))
    )
    info = extract_packet_info(pkt)
    assert info["protocol"] == "DNS"
    assert info["raw_summary"]["dns_query"] == "google.com."


def test_extract_payload_size():
    pkt = IP(src="10.0.0.1", dst="10.0.0.2") / TCP(sport=80, dport=5000) / Raw(load=b"x" * 100)
    info = extract_packet_info(pkt)
    assert info["payload_size"] == 100


def test_extract_non_ip_returns_none():
    from scapy.all import ARP
    pkt = ARP()
    info = extract_packet_info(pkt)
    assert info is None
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest tests/agent/test_scapy_capture.py -v
```

Expected: FAIL.

- [ ] **Step 3: Create `agent/capture/__init__.py`**

```python
```

- [ ] **Step 4: Create `agent/capture/scapy_capture.py`**

```python
import asyncio
import logging
from collections.abc import Callable

from scapy.all import IP, TCP, UDP, DNS, DNSQR, Raw, sniff

logger = logging.getLogger(__name__)


# Noise filter: skip these protocols/ports
NOISE_PORTS = {5353, 1900, 137, 138}  # mDNS, SSDP, NetBIOS
NOISE_IPS = {"224.0.0.251", "239.255.255.250", "255.255.255.255"}


def extract_packet_info(pkt) -> dict | None:
    """Extract structured info from a Scapy packet. Returns None for non-IP packets."""
    if not pkt.haslayer(IP):
        return None

    ip = pkt[IP]
    info = {
        "src_ip": ip.src,
        "dst_ip": ip.dst,
        "src_port": None,
        "dst_port": None,
        "protocol": ip.proto if isinstance(ip.proto, str) else "OTHER",
        "payload_size": 0,
        "flags": None,
        "direction": None,
        "raw_summary": None,
    }

    if pkt.haslayer(TCP):
        tcp = pkt[TCP]
        info["src_port"] = tcp.sport
        info["dst_port"] = tcp.dport
        info["protocol"] = "TCP"
        info["flags"] = str(tcp.flags)

    elif pkt.haslayer(UDP):
        udp = pkt[UDP]
        info["src_port"] = udp.sport
        info["dst_port"] = udp.dport
        info["protocol"] = "UDP"

        if pkt.haslayer(DNS) and pkt.haslayer(DNSQR):
            info["protocol"] = "DNS"
            qname = pkt[DNSQR].qname
            if isinstance(qname, bytes):
                qname = qname.decode("utf-8", errors="ignore")
            info["raw_summary"] = {"dns_query": qname}

    if pkt.haslayer(Raw):
        info["payload_size"] = len(pkt[Raw].load)

    return info


def is_noise(info: dict) -> bool:
    """Filter out noisy broadcast/multicast packets."""
    if info["dst_ip"] in NOISE_IPS:
        return True
    if info["src_port"] in NOISE_PORTS or info["dst_port"] in NOISE_PORTS:
        return True
    return False


class ScapyCapture:
    def __init__(self, callback: Callable[[dict], None], interface: str | None = None):
        self.callback = callback
        self.interface = interface
        self._running = False

    def _handle_packet(self, pkt):
        info = extract_packet_info(pkt)
        if info is None:
            return
        if is_noise(info):
            return
        self.callback(info)

    def start(self) -> None:
        """Start packet capture (blocking). Run in a thread."""
        self._running = True
        logger.info(f"Starting Scapy capture on interface: {self.interface or 'default'}")
        sniff(
            iface=self.interface,
            prn=self._handle_packet,
            store=False,
            stop_filter=lambda _: not self._running,
        )

    def stop(self) -> None:
        self._running = False
```

- [ ] **Step 5: Run capture tests**

```bash
pytest tests/agent/test_scapy_capture.py -v
```

Expected: 5 tests PASS.

- [ ] **Step 6: Write test for L1 Packet Engine**

`tests/core/test_packet_engine.py`:
```python
from datetime import datetime, timezone

from core.services.packet_engine import PacketEngine


def test_parse_packet_payload():
    engine = PacketEngine()
    payload = {
        "src_ip": "192.168.1.5",
        "dst_ip": "8.8.8.8",
        "src_port": 54321,
        "dst_port": 53,
        "protocol": "DNS",
        "payload_size": 64,
        "flags": None,
        "direction": "outbound",
        "raw_summary": {"dns_query": "google.com."},
    }
    packet_data = engine.parse_payload("phone-1", payload)
    assert packet_data["device_id"] == "phone-1"
    assert packet_data["src_ip"] == "192.168.1.5"
    assert packet_data["protocol"] == "DNS"
    assert "timestamp" in packet_data


def test_parse_packet_adds_timestamp():
    engine = PacketEngine()
    payload = {
        "src_ip": "10.0.0.1",
        "dst_ip": "10.0.0.2",
        "protocol": "TCP",
        "payload_size": 0,
    }
    data = engine.parse_payload("laptop", payload)
    ts = data["timestamp"]
    assert isinstance(ts, datetime)
```

- [ ] **Step 7: Create `core/services/packet_engine.py`**

```python
import logging
from datetime import datetime, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from core.models.packet import Packet

logger = logging.getLogger(__name__)


class PacketEngine:
    """L1 Packet Engine — parses raw packet payloads and stores them in TimescaleDB."""

    def parse_payload(self, device_id: str, payload: dict) -> dict:
        """Parse an agent packet payload into a dict ready for DB insertion."""
        return {
            "device_id": device_id,
            "timestamp": datetime.now(timezone.utc),
            "src_ip": payload.get("src_ip", "0.0.0.0"),
            "dst_ip": payload.get("dst_ip", "0.0.0.0"),
            "src_port": payload.get("src_port"),
            "dst_port": payload.get("dst_port"),
            "protocol": payload.get("protocol", "UNKNOWN"),
            "payload_size": payload.get("payload_size", 0),
            "flags": payload.get("flags"),
            "direction": payload.get("direction"),
            "flow_id": None,
            "raw_summary": payload.get("raw_summary"),
        }

    async def store_packet(self, db: AsyncSession, device_id: str, payload: dict) -> None:
        """Parse and store a single packet."""
        data = self.parse_payload(device_id, payload)
        packet = Packet(**data)
        db.add(packet)
        await db.commit()
        logger.debug(f"Stored packet: {data['src_ip']} -> {data['dst_ip']} ({data['protocol']})")

    async def store_batch(self, db: AsyncSession, device_id: str, payloads: list[dict]) -> int:
        """Parse and store a batch of packets. Returns count stored."""
        for payload in payloads:
            data = self.parse_payload(device_id, payload)
            db.add(Packet(**data))
        await db.commit()
        return len(payloads)
```

- [ ] **Step 8: Run packet engine tests**

```bash
pytest tests/core/test_packet_engine.py -v
```

Expected: 2 tests PASS.

- [ ] **Step 9: Commit**

```bash
git add agent/capture/ core/services/packet_engine.py tests/agent/test_scapy_capture.py tests/core/test_packet_engine.py
git commit -m "feat: Scapy packet capture + L1 Packet Engine for parsing and storage"
```

---

## Task 12: Wire Everything Together — End-to-End Integration

**Files:**
- Modify: `core/api/agent_ws.py` (add packet engine integration)
- Modify: `agent/main.py` (add capture integration)

- [ ] **Step 1: Update `core/api/agent_ws.py` to store packets**

Replace the full file:
```python
import logging

from fastapi import APIRouter, WebSocket, WebSocketDisconnect
from redis.asyncio import Redis

from core.config import settings
from core.database import async_session
from core.redis_client import get_redis
from core.services.agent_manager import AgentManager
from core.services.packet_engine import PacketEngine
from core.schemas.agent_protocol import MessageType
from core.models.device import Device

from sqlalchemy import select

logger = logging.getLogger(__name__)

router = APIRouter()
packet_engine = PacketEngine()


@router.websocket("/ws/agent")
async def agent_websocket(websocket: WebSocket):
    key = websocket.query_params.get("key", "")
    if not key:
        key = websocket.headers.get("x-agent-key", "")

    redis = await get_redis()
    manager = AgentManager(redis=redis, agent_secret_key=settings.agent_secret_key)

    if not manager.verify_key(key):
        await websocket.close(code=4001, reason="Unauthorized")
        await redis.aclose()
        return

    await websocket.accept()

    device_id = None
    try:
        while True:
            raw = await websocket.receive_bytes()
            msg = manager.parse_message(raw)
            if msg is None:
                continue

            if device_id is None:
                device_id = msg.device_id
                await manager.register_connection(device_id, websocket)
                # Auto-register device in DB if new
                async with async_session() as db:
                    result = await db.execute(select(Device).where(Device.id == device_id))
                    if not result.scalar_one_or_none():
                        device = Device(
                            id=device_id,
                            name=device_id,
                            device_type="unknown",
                            status="online",
                        )
                        db.add(device)
                        await db.commit()
                        logger.info(f"Auto-registered new device: {device_id}")

            if msg.type == MessageType.HEARTBEAT:
                await manager.handle_heartbeat(msg.device_id, msg.payload)
                # Update device in DB
                async with async_session() as db:
                    result = await db.execute(select(Device).where(Device.id == msg.device_id))
                    device = result.scalar_one_or_none()
                    if device:
                        device.status = "online"
                        device.network_type = msg.payload.get("network_type")
                        device.local_ip = msg.payload.get("local_ip")
                        device.agent_version = msg.payload.get("agent_version")
                        device.os_info = msg.payload.get("os_info")
                        from datetime import datetime, timezone
                        device.last_seen = datetime.now(timezone.utc)
                        await db.commit()

            elif msg.type == MessageType.PACKET_FLAG:
                async with async_session() as db:
                    await packet_engine.store_packet(db, msg.device_id, msg.payload)

            elif msg.type == MessageType.BULK_STATS:
                if "packets" in msg.payload:
                    async with async_session() as db:
                        await packet_engine.store_batch(
                            db, msg.device_id, msg.payload["packets"]
                        )

    except WebSocketDisconnect:
        if device_id:
            await manager.unregister_connection(device_id)
            async with async_session() as db:
                result = await db.execute(select(Device).where(Device.id == device_id))
                device = result.scalar_one_or_none()
                if device:
                    device.status = "offline"
                    await db.commit()
    finally:
        await redis.aclose()
```

- [ ] **Step 2: Update `agent/main.py` to integrate Scapy capture**

Replace the `_run_agent` function and add imports:

```python
import asyncio
import logging
import platform
import threading

import typer
from rich.console import Console
from rich.table import Table

from agent.config import AgentConfig, DEFAULT_CONFIG_PATH
from agent.transport import AgentTransport
from agent.capture.scapy_capture import ScapyCapture

app = typer.Typer(name="netsen-agent", help="NetSentinel Device Agent")
console = Console()


@app.command()
def setup(
    server: str = typer.Option(..., help="Core server WebSocket URL"),
    name: str = typer.Option(..., help="Device name"),
    key: str = typer.Option(..., help="Agent API key"),
    capture_mode: str = typer.Option("scapy", help="Capture mode: scapy, tcpdump, vpn-tun"),
):
    """Configure the agent for this device."""
    config = AgentConfig(
        server_url=server,
        device_name=name,
        api_key=key,
        capture_mode=capture_mode,
    )
    config.save()
    console.print(f"[green]Agent configured![/green]")
    console.print(f"  Device ID: {config.device_id}")
    console.print(f"  Server: {config.server_url}")
    console.print(f"  Config saved to: {DEFAULT_CONFIG_PATH}")


@app.command()
def start():
    """Start the agent."""
    try:
        config = AgentConfig.load()
    except FileNotFoundError:
        console.print("[red]No config found. Run 'netsen-agent setup' first.[/red]")
        raise typer.Exit(1)

    console.print(f"[green]Starting NetSentinel Agent[/green]")
    console.print(f"  Device: {config.device_name} ({config.device_id})")
    console.print(f"  Server: {config.server_url}")
    console.print(f"  Capture: {config.capture_mode}")

    asyncio.run(_run_agent(config))


async def _run_agent(config: AgentConfig):
    transport = AgentTransport(config)
    packet_queue: asyncio.Queue = asyncio.Queue()

    def on_packet(info: dict):
        """Callback from Scapy capture thread — puts packet into async queue."""
        try:
            packet_queue.put_nowait(info)
        except asyncio.QueueFull:
            pass  # Drop packet if queue is full

    try:
        await transport.connect()
        console.print("[green]Connected to Core![/green]")

        # Send initial heartbeat
        await transport.send_heartbeat()

        # Start capture in a background thread
        capture = ScapyCapture(callback=on_packet)
        capture_thread = threading.Thread(target=capture.start, daemon=True)
        capture_thread.start()
        console.print("[green]Packet capture started![/green]")

        # Run heartbeat and packet sender concurrently
        await asyncio.gather(
            transport.run_heartbeat_loop(),
            _packet_sender(transport, packet_queue),
        )

    except KeyboardInterrupt:
        console.print("\n[yellow]Shutting down...[/yellow]")
    except Exception as e:
        console.print(f"[red]Error: {e}[/red]")
    finally:
        transport.stop()
        await transport.disconnect()


async def _packet_sender(transport: AgentTransport, queue: asyncio.Queue):
    """Reads packets from queue and sends to Core."""
    while True:
        info = await queue.get()
        try:
            await transport.send_packet(**info)
        except Exception as e:
            logging.error(f"Failed to send packet: {e}")
            break


@app.command()
def status():
    """Show agent status."""
    try:
        config = AgentConfig.load()
        table = Table(title="Agent Status")
        table.add_column("Field", style="cyan")
        table.add_column("Value", style="green")
        table.add_row("Device Name", config.device_name)
        table.add_row("Device ID", config.device_id)
        table.add_row("Server", config.server_url)
        table.add_row("Capture Mode", config.capture_mode)
        table.add_row("Heartbeat Interval", f"{config.heartbeat_interval}s")
        console.print(table)
    except FileNotFoundError:
        console.print("[red]No config found. Run 'netsen-agent setup' first.[/red]")


if __name__ == "__main__":
    app()
```

- [ ] **Step 3: Run all tests to make sure nothing is broken**

```bash
pytest tests/ -v
```

Expected: All tests PASS.

- [ ] **Step 4: End-to-end manual test**

Terminal 1 — Start infrastructure:
```bash
docker-compose up -d
alembic upgrade head
uvicorn core.main:app --reload --port 8000
```

Terminal 2 — Start agent:
```bash
netsen-agent setup --server ws://localhost:8000/ws/agent --name test-laptop --key change_me_to_random_agent_key
sudo netsen-agent start
```

Terminal 3 — Verify:
```bash
# Check device registered
curl http://localhost:8000/api/devices | python -m json.tool

# Wait 30s for heartbeat, then check again
curl http://localhost:8000/api/devices | python -m json.tool

# Generate some traffic (open a browser), then check packets
curl "http://localhost:8000/api/devices/test-laptop-XXXX/packets?since_minutes=5" | python -m json.tool
```

Expected: Device shows as online, packets appear in the API response.

- [ ] **Step 5: Commit**

```bash
git add core/api/agent_ws.py agent/main.py
git commit -m "feat: end-to-end integration — agent captures packets, streams to Core, stored in DB, queryable via API"
```

---

## Task 13: Final Cleanup + Phase 1 Milestone Commit

- [ ] **Step 1: Run full test suite**

```bash
pytest tests/ -v --tb=short
```

Expected: All tests PASS.

- [ ] **Step 2: Verify Docker Compose health**

```bash
docker-compose ps
```

Expected: All services healthy.

- [ ] **Step 3: Update README roadmap**

In `README.md`, change:
```
- [ ] Phase 1: Core server + agent + packet capture
```
to:
```
- [x] Phase 1: Core server + agent + packet capture
```

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "milestone: Phase 1 complete — Core server, agent, packet capture, end-to-end pipeline"
git push
```

---

## Summary

| Task | What It Builds | Key Files |
|------|---------------|-----------|
| 1 | Project skeleton + deps | `pyproject.toml`, `core/config.py` |
| 2 | Docker Compose | `docker-compose.yml` |
| 3 | Database + models + migrations | `core/database.py`, `core/models/`, `alembic.ini` |
| 4 | Redis device state | `core/redis_client.py` |
| 5 | FastAPI + health | `core/main.py`, `core/api/health.py` |
| 6 | REST API (devices + packets) | `core/api/devices.py`, `core/api/packets.py` |
| 7 | Agent protocol schemas | `core/schemas/agent_protocol.py` |
| 8 | WebSocket agent manager | `core/services/agent_manager.py` |
| 9 | WebSocket endpoint | `core/api/agent_ws.py` |
| 10 | Agent CLI + config + transport | `agent/main.py`, `agent/config.py`, `agent/transport.py` |
| 11 | Scapy capture + L1 engine | `agent/capture/scapy_capture.py`, `core/services/packet_engine.py` |
| 12 | End-to-end wiring | Integration of all components |
| 13 | Cleanup + milestone | Tests pass, pushed |
