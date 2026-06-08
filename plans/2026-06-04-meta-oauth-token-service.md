# Meta (Facebook/Instagram) OAuth + Token Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let an agency connect its Meta account via OAuth, store Facebook Page + linked Instagram tokens (base64-obfuscated) in one flat table, and keep the long-lived user token alive with a daily refresh + expiry-warning job.

**Architecture:** FastAPI backend (`src/`) + React/TS frontend (`frontend/`). One flat `agency_social_connections` table (one row per Page/IG; per-agency user-token/expiry/warning fields carried on the rows). A thin in-repo `MetaGraphClient` (httpx, no SDK). OAuth handshake under `/api/auth/meta/*` (unauthenticated callback uses an HMAC-signed `state` to resolve tenant+agency without a DB scan); management under `/api/social/meta/*`. A daily `asyncio` loop in `lifespan` refreshes/ warns per agency.

**Tech Stack:** Python 3 / FastAPI / SQLAlchemy async / Alembic / httpx / structlog; React 19 / TypeScript / Vite / TanStack Query v5 / Tailwind v4.

**Design spec:** `/Users/binhquach/Workplace/Slicify AI/slicify-docs/specs/2026-06-04-meta-oauth-token-service-design.md`

**Testing policy (from the approved spec):** Automated **integration tests only for the daily background job** (Milestone 3). Milestones 1, 2, 4 are verified manually (run the app, click Connect, observe DB rows) — do **not** write unit tests for the OAuth endpoints or frontend in this phase.

**Sequencing:** UI-first. Milestone 1 delivers a clickable end-to-end Facebook connect that lands rows in the DB. Then Instagram (M2), the daily monitor + tests (M3), and the reconnect notification (M4).

---

## File structure

**Backend (create):**
- `src/models/agency_social_connection.py` — the ORM model (one responsibility: the table).
- `src/integrations/meta_graph.py` — `MetaGraphClient` SDK-lite + `MetaGraphError`.
- `src/api/meta_social.py` — all HTTP endpoints (connect/callback/connections/disconnect) + the obfuscate + signed-state helpers + `DEFAULT_AGENCY_ID`.
- `src/services/meta_token_monitor.py` — daily job: per-agency refresh/warn logic + the loop.
- `alembic/versions/<rev>_add_agency_social_connections.py` — migration.
- `tests/integration/test_meta_token_monitor.py` — job tests.

**Backend (modify):**
- `src/config.py` — add `META_GRAPH_VERSION`.
- `src/models/__init__.py` — register the new model on `Base.metadata`.
- `src/main.py` — include the two new routers + start the daily loop in `lifespan`.
- `src/db/repositories/user.py` — add `get_admins()`.

**Frontend (create):**
- `frontend/src/api/metaCredentials.ts` — API calls.
- `frontend/src/pages/MetaSettingsPage.tsx` — `MetaSettingsPanel` (fetches + orchestrates).
- `frontend/src/components/integrations/MetaAccountCard.tsx` — presentational card.
- `frontend/src/components/integrations/MetaStatusBanner.tsx` — in-app reconnect/expiry banner (driven by `connection.status`; replaces the email reconnect notice — see Task 10/Milestone 4 as-built).
- `frontend/src/lib/metaHealth.ts` — **(M5)** pure `deriveMetaHealth(connection)` + `daysUntil`/`formatConnectionDate` display helpers. Maps `token_expires_at` (real-time day math) + backend `status` to a single `MetaHealth` state consumed by both the card and the banner.

**Frontend (modify):**
- `frontend/src/types/meta.ts` — `MetaConnection`, `MetaConnectResponse`; **(M5)** `MetaHealthLevel`, `MetaHealth`.
- `frontend/src/pages/IntegrationsPage.tsx` — add `meta` to `FIXED_TABS` + dispatch the panel.
- **(M5)** `MetaAccountCard.tsx` — render the connected account name + connection date + token expiry, health-driven status pill (green/amber/red), and a prominent (red) reconnect CTA in the imminent/expired/needs-reconnect states.
- **(M5)** `MetaStatusBanner.tsx` — switch from raw `status` to the derived `MetaHealth`: amber for `expiring` (≤14d), red + reconnect button for `imminent` (≤5d) / `expired` / `needs_reconnect`.
- **(M5)** `MetaSettingsPage.tsx` — compute `deriveMetaHealth(connection)` once and pass per-card detail (name/date/expiry) + health to both cards and the banner.

**Constants used across files:** `DEFAULT_AGENCY_ID = "default"` (server-side resolved agency id for v1), `META_OAUTH_COMPLETE = "meta-oauth-complete"` (postMessage string).

---

## Milestone 1 — Testable vertical slice: connect Facebook → store rows

### Task 1: Add `META_GRAPH_VERSION` to config

**Files:**
- Modify: `src/config.py` (after `FACEBOOK_APP_SECRET`, currently line 47)

- [ ] **Step 1: Add the field**

In `class Settings(BaseSettings)`, immediately after the `FACEBOOK_APP_SECRET` line, add:

```python
    META_GRAPH_VERSION: str = "v21.0"
```

- [ ] **Step 2: Verify it loads**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.config import settings; print(settings.META_GRAPH_VERSION)"`
Expected: `v21.0`

- [ ] **Step 3: Commit**

```bash
git add src/config.py
git commit -m "feat(meta): add META_GRAPH_VERSION setting"
```

---

### Task 2: Create the `AgencySocialConnection` + `AgencySocialAccount` models

> **As-built (normalized):** the per-agency **user-token lifecycle** lives in `agency_social_connections` (one row per agency); each Page/IG is a child row in `agency_social_accounts` (its own permanent page token), joined by a `connection.accounts` relationship. This replaced the earlier single flat table that duplicated the user token across every Page row. See spec §4.

**Files:**
- Create: `src/models/agency_social_connection.py`
- Create: `src/models/agency_social_account.py`
- Modify: `src/models/__init__.py`

- [ ] **Step 1: Write the connection model (the OAuth login — one per agency)**

Create `src/models/agency_social_connection.py`:

```python
import uuid
from datetime import datetime
from typing import TYPE_CHECKING, Optional

from sqlalchemy import (
    DateTime,
    ForeignKey,
    SmallInteger,
    String,
    Text,
    UniqueConstraint,
)
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from src.models.base import Base, TimestampMixin, UUIDMixin

if TYPE_CHECKING:
    from src.models.agency_social_account import AgencySocialAccount


class AgencySocialConnection(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "agency_social_connections"
    __table_args__ = (
        UniqueConstraint("agency_id", name="uq_agency_social_connections_agency"),
    )

    agency_id: Mapped[str] = mapped_column(String(100), nullable=False, index=True)
    user_access_token: Mapped[Optional[str]] = mapped_column(Text)
    token_expires_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True)
    )
    status: Mapped[str] = mapped_column(
        String(20), nullable=False, server_default="connected"
    )
    warning_level: Mapped[int] = mapped_column(
        SmallInteger, nullable=False, server_default="0"
    )
    last_error: Mapped[Optional[str]] = mapped_column(Text)
    connected_by_user_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    connected_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    last_refreshed_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True)
    )
    last_warned_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))

    accounts: Mapped[list["AgencySocialAccount"]] = relationship(
        "AgencySocialAccount",
        back_populates="connection",
        cascade="all, delete-orphan",
        lazy="selectin",
    )
```

(Each column carries a short `comment=` in the as-built file; omitted here for brevity.)

- [ ] **Step 2: Write the account model (one per Page/IG)**

Create `src/models/agency_social_account.py`:

```python
import uuid
from datetime import datetime
from typing import TYPE_CHECKING, Optional

from sqlalchemy import (
    Boolean,
    DateTime,
    ForeignKey,
    Index,
    String,
    Text,
    UniqueConstraint,
)
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from src.models.base import Base, TimestampMixin, UUIDMixin

if TYPE_CHECKING:
    from src.models.agency_social_connection import AgencySocialConnection


class AgencySocialAccount(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "agency_social_accounts"
    __table_args__ = (
        UniqueConstraint(
            "connection_id",
            "platform",
            "provider_id",
            name="uq_agency_social_accounts_provider",
        ),
        Index("ix_agency_social_accounts_agency_id", "agency_id"),
    )

    connection_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("agency_social_connections.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    agency_id: Mapped[str] = mapped_column(String(100), nullable=False)
    platform: Mapped[str] = mapped_column(String(20), nullable=False)
    provider_id: Mapped[str] = mapped_column(String(100), nullable=False)
    name: Mapped[Optional[str]] = mapped_column(String(255))
    username: Mapped[Optional[str]] = mapped_column(String(255))
    access_token: Mapped[Optional[str]] = mapped_column(Text)
    is_primary: Mapped[bool] = mapped_column(
        Boolean, nullable=False, server_default="false"
    )
    authorized: Mapped[bool] = mapped_column(
        Boolean, nullable=False, server_default="true"
    )
    connected_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))

    connection: Mapped["AgencySocialConnection"] = relationship(
        "AgencySocialConnection", back_populates="accounts"
    )
```

- [ ] **Step 3: Register both models so `Base.metadata` includes the tables**

In `src/models/__init__.py`, add (alphabetical, before the connection import) and to `__all__`:

```python
from src.models.agency_social_account import AgencySocialAccount
from src.models.agency_social_connection import AgencySocialConnection
```

- [ ] **Step 4: Verify the models import, register, and map cleanly**

Run (Docker backend; mappers must resolve the relationship):
`docker compose -f docker-compose.local.yml run --rm --no-deps backend python -c "from sqlalchemy.orm import configure_mappers; from src.models import Base, AgencySocialAccount, AgencySocialConnection; configure_mappers(); print({'agency_social_connections','agency_social_accounts'} <= set(Base.metadata.tables))"`
Expected: `True`

- [ ] **Step 5: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 3: Alembic migration for `agency_social_connections`

**Files:**
- Create: `alembic/versions/m1a2b3c4d5e6_add_agency_social_connections.py`

> **Local-environment realities (discovered during execution — read before running anything):**
>
> 1. **The venv is `.venv/`, and the `alembic` console script has a stale shebang** (points at the old `/Volumes/T7/...` path). Do **not** run `venv/bin/alembic` or `.venv/bin/alembic`. Run alembic **inside the Docker backend container** instead — that's the canonical local path and where `.env.local` (creds `slicify`/`slicify`, `DB_HOST=postgres`) is mounted as `/app/.env`: `docker compose -f docker-compose.local.yml run --rm --no-deps backend alembic <cmd>`
> 2. **The local dev DB is from `docker-compose.local.yml`** — Postgres 16, db `sideline_realestate`, user/pass `slicify`/`slicify` on `localhost:5432`. The repo's `.env` (user `ab5`, empty password) is **not** the local config; `.env.local` is. For host-side `psql`: `PGPASSWORD=slicify psql -h localhost -U slicify -d sideline_realestate`.
> 3. **There were two alembic heads** (`j1e2f3g4h5i6` add_sms_templates [a merge], `k1l2m3n4o5p6` add_swot) both descending from `j1b2c3d4e5f6`. This migration **merges them** via a tuple `down_revision` — that's why it isn't chained off a single revision.
> 4. **The local DB's migration chain is inconsistent** (`alembic current` = `ffa04a9385ae`, but the schema already contains tables from much later/parallel migrations like `support_tickets` while missing earlier ones like `appraisal_charities`). `alembic upgrade head` will therefore **fail** locally (it re-creates already-present tables). This is a pre-existing local-env condition, not caused by this feature. For **local dev** we create just this one table directly from the model metadata (Step 3). The migration file is still authoritative for **clean tenant/template DBs** at deploy time.
> 5. **As-built after the `origin/dev` sync (120-commit merge):** the two original parents `j1e2f3g4h5i6`/`k1l2m3n4o5p6` are both already ancestors of dev's head `e3w4x5y6z7a8` (dev merged them at `b6782e0f6c55`), so the committed `down_revision` was **rebased onto `e3w4x5y6z7a8`** (a single value, not the tuple shown in the Step 1 block) per the CLAUDE.md multi-head rule. That collapses alembic back to one head (`m1a2b3c4d5e6` → descends from `e3w4x5y6z7a8`). The Step 1 code block keeps the original tuple for historical context; the on-disk file now reads `down_revision = "e3w4x5y6z7a8"`.

- [ ] **Step 1: Write the migration (two tables, up + down)**

Create `alembic/versions/m1a2b3c4d5e6_add_agency_social_connections.py`. Columns/nullability/`server_default`s must match the two models in Task 2. `upgrade()` creates `agency_social_connections` first, then `agency_social_accounts` (FK → connection, `ON DELETE CASCADE`); `downgrade()` drops accounts then connections.

```python
"""Add agency_social_connections + agency_social_accounts (Meta OAuth)

Revision ID: m1a2b3c4d5e6
Revises: j1e2f3g4h5i6, k1l2m3n4o5p6
Create Date: 2026-06-04
"""

from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "m1a2b3c4d5e6"
down_revision: Union[str, Sequence[str], None] = ("j1e2f3g4h5i6", "k1l2m3n4o5p6")
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.create_table(
        "agency_social_connections",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("agency_id", sa.String(length=100), nullable=False),
        sa.Column("user_access_token", sa.Text(), nullable=True),
        sa.Column("token_expires_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column(
            "status", sa.String(length=20), nullable=False, server_default="connected"
        ),
        sa.Column(
            "warning_level", sa.SmallInteger(), nullable=False, server_default="0"
        ),
        sa.Column("last_error", sa.Text(), nullable=True),
        sa.Column("connected_by_user_id", postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column("connected_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("last_refreshed_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("last_warned_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.ForeignKeyConstraint(["connected_by_user_id"], ["users.id"]),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("agency_id", name="uq_agency_social_connections_agency"),
    )
    op.create_index(
        "ix_agency_social_connections_agency_id",
        "agency_social_connections",
        ["agency_id"],
    )
    op.create_table(
        "agency_social_accounts",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("connection_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("agency_id", sa.String(length=100), nullable=False),
        sa.Column("platform", sa.String(length=20), nullable=False),
        sa.Column("provider_id", sa.String(length=100), nullable=False),
        sa.Column("name", sa.String(length=255), nullable=True),
        sa.Column("username", sa.String(length=255), nullable=True),
        sa.Column("access_token", sa.Text(), nullable=True),
        sa.Column(
            "is_primary", sa.Boolean(), nullable=False, server_default="false"
        ),
        sa.Column(
            "authorized", sa.Boolean(), nullable=False, server_default="true"
        ),
        sa.Column("connected_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.ForeignKeyConstraint(
            ["connection_id"],
            ["agency_social_connections.id"],
            ondelete="CASCADE",
        ),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint(
            "connection_id",
            "platform",
            "provider_id",
            name="uq_agency_social_accounts_provider",
        ),
    )
    op.create_index(
        "ix_agency_social_accounts_connection_id",
        "agency_social_accounts",
        ["connection_id"],
    )
    op.create_index(
        "ix_agency_social_accounts_agency_id",
        "agency_social_accounts",
        ["agency_id"],
    )


def downgrade() -> None:
    op.drop_index(
        "ix_agency_social_accounts_agency_id", table_name="agency_social_accounts"
    )
    op.drop_index(
        "ix_agency_social_accounts_connection_id", table_name="agency_social_accounts"
    )
    op.drop_table("agency_social_accounts")
    op.drop_index(
        "ix_agency_social_connections_agency_id",
        table_name="agency_social_connections",
    )
    op.drop_table("agency_social_connections")
```

- [ ] **Step 2: Confirm the migration collapses the two heads into one**

Run: `docker compose -f docker-compose.local.yml run --rm --no-deps backend alembic heads`
Expected: a single head — `m1a2b3c4d5e6 (head)`.

- [ ] **Step 3: Apply to the local dev DB (drop the old flat table, create the two new ones)**

The local alembic chain is inconsistent (realities #4 above), so don't `alembic upgrade head`. Drop the prior single table and create the two new tables from model metadata (dependency order: connection then account):

```bash
docker compose -f docker-compose.local.yml run --rm --no-deps backend python -c "
from sqlalchemy import create_engine, text
from src.config import settings
from src.models.agency_social_connection import AgencySocialConnection
from src.models.agency_social_account import AgencySocialAccount
url = f'postgresql://{settings.DB_USER}:{settings.DB_PASSWORD}@{settings.DB_HOST}:{settings.DB_PORT}/{settings.DB_DATABASE}'
e = create_engine(url)
with e.begin() as c:
    c.execute(text('DROP TABLE IF EXISTS agency_social_accounts CASCADE'))
    c.execute(text('DROP TABLE IF EXISTS agency_social_connections CASCADE'))
AgencySocialConnection.__table__.create(bind=e, checkfirst=True)
AgencySocialAccount.__table__.create(bind=e, checkfirst=True)
print('ok')
"
```

Expected: `ok`. (Drops the old flat table — any existing dev connection is cleared; reconnect afterwards.)

- [ ] **Step 4: Verify the migration up/down/up + both tables**

The migration's `upgrade()`/`downgrade()` were verified against the local DB by binding the alembic `op` proxy and running upgrade → downgrade → upgrade; both tables create and drop cleanly and the columns match the models. To spot-check the live tables:

`PGPASSWORD=slicify psql -h localhost -U slicify -d sideline_realestate -c "\d agency_social_connections" -c "\d agency_social_accounts"`
Expected: connections has `UNIQUE(agency_id)` + FK→`users`; accounts has `UNIQUE(connection_id, platform, provider_id)` + FK→`agency_social_connections` `ON DELETE CASCADE` + index on `agency_id`/`connection_id`.

- [ ] **Step 5: (No commit)** — leave the migration in the working tree for review.

> **Deploy execution (not local):** on clean tenant DBs + `slate_template`, the migration applies normally via `scripts/migrate_all_tenants.py` — it merges the two heads and creates both tables in one pass. The direct metadata-create above is **local-dev only**.

---

### Task 4: `MetaGraphClient` (SDK-lite)

**Files:**
- Create: `src/integrations/meta_graph.py`

> **As-built (expires_in robustness):** Facebook's long-lived token exchange does **not** always include `expires_in` (it's omitted/`0` when re-authorizing an already-authorized app, or for effectively non-expiring tokens). Reading it as a hard `data["expires_in"]` raised `KeyError` → 500 on the callback during **reconnect**. `exchange_for_long_lived` defaults a missing/zero value to `LONG_LIVED_TTL_SECONDS` (60 days, Facebook's standard long-lived user-token lifetime). `refresh_long_lived` reuses the same path, so the monitor inherits the safety too.

- [ ] **Step 1: Write the client**

Create `src/integrations/meta_graph.py`:

Return types are precise `TypedDict`s derived from the Meta Graph API docs (token endpoints return `access_token`/`expires_in`/`token_type`; `/me/accounts` returns page objects with `id`/`name`/`access_token`; the `instagram_business_account` subfield returns `id`/`username`). `_get` stays the untyped JSON boundary (`dict[str, Any]`) so the public methods narrow without any `cast()` — `.get(...)` yields `Any`, which is assignable to the declared `TypedDict` returns:

```python
from typing import Any, NotRequired, Optional, TypedDict

import httpx
import structlog

from src.config import settings

logger = structlog.get_logger()

LONG_LIVED_TTL_SECONDS = 60 * 24 * 60 * 60


class MetaTokenResponse(TypedDict):
    access_token: str
    expires_in: int
    token_type: NotRequired[str]


class MetaPage(TypedDict):
    id: str
    name: str
    access_token: str


class MetaIgAccount(TypedDict):
    id: str
    username: NotRequired[str]


class MetaGraphError(Exception):
    pass


class MetaGraphClient:
    def __init__(self) -> None:
        self.app_id = settings.FACEBOOK_APP_ID
        self.app_secret = settings.FACEBOOK_APP_SECRET
        self.base_url = f"https://graph.facebook.com/{settings.META_GRAPH_VERSION}"

    async def _get(self, path: str, params: dict[str, Any]) -> dict[str, Any]:
        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(f"{self.base_url}{path}", params=params)
        if resp.status_code != 200:
            logger.warning(
                "meta_graph.error", path=path, status=resp.status_code, body=resp.text
            )
            raise MetaGraphError(f"Meta Graph {path} -> {resp.status_code}: {resp.text}")
        return resp.json()

    async def exchange_code(self, code: str, redirect_uri: str) -> str:
        data = await self._get(
            "/oauth/access_token",
            {
                "client_id": self.app_id,
                "client_secret": self.app_secret,
                "redirect_uri": redirect_uri,
                "code": code,
            },
        )
        return data["access_token"]

    async def exchange_for_long_lived(self, short_token: str) -> MetaTokenResponse:
        data = await self._get(
            "/oauth/access_token",
            {
                "grant_type": "fb_exchange_token",
                "client_id": self.app_id,
                "client_secret": self.app_secret,
                "fb_exchange_token": short_token,
            },
        )
        result: MetaTokenResponse = {
            "access_token": data["access_token"],
            "expires_in": int(data.get("expires_in") or LONG_LIVED_TTL_SECONDS),
        }
        if "token_type" in data:
            result["token_type"] = data["token_type"]
        return result

    async def refresh_long_lived(self, long_token: str) -> MetaTokenResponse:
        return await self.exchange_for_long_lived(long_token)

    async def get_pages(self, user_token: str) -> list[MetaPage]:
        data = await self._get(
            "/me/accounts",
            {"access_token": user_token, "fields": "id,name,access_token"},
        )
        return data.get("data", [])

    async def get_ig_account(
        self, page_id: str, page_token: str
    ) -> Optional[MetaIgAccount]:
        data = await self._get(
            f"/{page_id}",
            {
                "access_token": page_token,
                "fields": "instagram_business_account{id,username}",
            },
        )
        return data.get("instagram_business_account")
```

> **Task 5 compatibility:** the `TypedDict`s are plain dicts at runtime, so downstream subscripting (`long["access_token"]`, `long["expires_in"]`, `page["id"]`, `ig["id"]`) is unchanged. `token_type` and `ig["username"]` are `NotRequired` — read them with `.get(...)`.

- [ ] **Step 2: Verify it imports**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/python -c "from src.integrations.meta_graph import MetaGraphClient, MetaGraphError, MetaTokenResponse, MetaPage, MetaIgAccount; print(MetaGraphClient().base_url)"`
Expected: `https://graph.facebook.com/v21.0` (note: the venv is `.venv/`, not `venv/`).

- [ ] **Step 3: (No commit)** — leave the file in the working tree for review per the execution override.

---

### Task 5: Backend API — connect / callback / connection / disconnect

**Files:**
- Create: `src/api/meta_social.py`
- Modify: `src/main.py`

- [ ] **Step 1: Write the API module**

> **As-built note (authoritative) — implemented against the two-table model (Task 2) and split across strict layers (per `src/CLAUDE.md`). The long Python block further down is the ORIGINAL flat-table sketch, kept only for narrative; it is SUPERSEDED. Source of truth is the actual files described here:**
>
> - **Router** `src/api/meta_social.py` — thin endpoints only: DI + call service + return schema. Keeps `_resolve_slug` (request concern) and `_close_window_html` (OAuth-popup presentation, returns `HTMLResponse`). `connect` needs no DB. Authenticated endpoints (`connect/instagram`, `connection` GET/DELETE) use `Depends(get_tenant_scoped_db)`; the unauthenticated `callback` resolves the tenant from the signed `state`, opens `get_tenant_session_factory(slug)`, calls `service.connect_pages`, commits. The "connect Facebook first" check is a `409` raised in the router off `service.has_facebook()`. `GET /connection` returns `MetaConnectionResponse | None`; `DELETE /connection` → 204.
> - **Service** `src/services/meta_social_service.py` — module-level OAuth mechanics (`make_state`/`parse_state`/`facebook_authorize_url`/`instagram_authorize_url`/`redirect_uri`/`_expiry_from`, plus `FACEBOOK_SCOPES`/`INSTAGRAM_SCOPES`/`STATE_TTL_SECONDS`/`WARNING_LEVEL_NONE`) **stateless, no session**; `parse_state` raises `ValueError` (never `HTTPException`). Class `MetaSocialService(session, graph_client=None)` **injects** `MetaGraphClient` with a default fallback (`self.graph = graph_client or MetaGraphClient()`, matching `daily_briefing`/`reengagement_engine`/`phone_call_processor`) so tests can pass a fake. Methods: `has_facebook` (connection exists AND has a facebook account), `get_connection`, `disconnect`, `connect_pages` (exchange → `_store`). `_store` deletes the agency's connection (the DB FK `ON DELETE CASCADE` clears its accounts), builds **one** `AgencySocialConnection` (user token + lifecycle) and appends `AgencySocialAccount` children via `_attach_accounts` (one facebook account per page + an instagram account when the page has a linked IG business account); `is_primary` = first page. Row builders: `_connection_row`/`_facebook_account`/`_instagram_account` (page token obfuscated onto each account; IG reuses the page token). `MetaGraphError` propagates to the router error popup.
> - **Repository** `src/db/repositories/agency_social_connection.py` — `AgencySocialConnectionRepository(BaseRepository)`: `get_by_agency` (returns the single connection; its `accounts` are eager-loaded via the relationship's `lazy="selectin"`) and `delete_by_agency`. Accounts need **no** separate repo — reads come through the relationship, writes through the cascade.
> - **Schema** `src/schemas/meta_social.py` — `AuthorizeUrlResponse`; `MetaAccountResponse` (per Page/IG asset: `platform`/`provider_id`/`name`/`username`/`is_primary`/`authorized`); `MetaConnectionResponse` (one per agency: `status`/`token_expires_at`/`warning_level`/`connected_at` + nested `accounts: list[MetaAccountResponse]`). Both ORM schemas use `model_config = {"from_attributes": True}`; the GET endpoint returns `MetaConnectionResponse.model_validate(row)` or `None`.
> - **Constants** — `MetaPlatform` and `MetaConnectionStatus` str-enums in `src/constants.py` (shared by service; assigned via `.value` to the String columns).
> - **Obfuscation** — shared `src/utils.py` `obfuscate`/`deobfuscate` (base64 for now; swap to real encryption there later), imported by the service.

Create `src/api/meta_social.py` (⚠️ flat-table sketch — superseded by the two-table as-built note above):

```python
import base64
import hashlib
import hmac
import json
import secrets
import time
from datetime import datetime, timezone
from typing import Optional

import structlog
from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import HTMLResponse
from sqlalchemy import delete, select, update
from sqlalchemy.ext.asyncio import AsyncSession

from src.api.deps import get_current_user, require_permission
from src.config import settings
from src.db.session import get_tenant_session_factory
from src.integrations.meta_graph import MetaGraphClient, MetaGraphError
from src.models.agency_social_connection import AgencySocialConnection
from src.models.user import User

logger = structlog.get_logger()

DEFAULT_AGENCY_ID = "default"
META_OAUTH_COMPLETE = "meta-oauth-complete"
STATE_TTL_SECONDS = 600
FACEBOOK_SCOPES = [
    "pages_show_list",
    "pages_read_engagement",
    "business_management",
]
INSTAGRAM_SCOPES = ["instagram_basic"]

auth_router = APIRouter(prefix="/api/auth/meta", tags=["meta-oauth"])
data_router = APIRouter(
    prefix="/api/social/meta",
    tags=["meta-social"],
    dependencies=[Depends(require_permission("settings:manage"))],
)


def _obfuscate(value: str) -> str:
    return base64.b64encode(value.encode()).decode()


def _deobfuscate(encoded: str) -> str:
    return base64.b64decode(encoded.encode()).decode()


def _make_state(slug: str, agency_id: str) -> str:
    payload = {
        "slug": slug,
        "agency_id": agency_id,
        "exp": int(time.time()) + STATE_TTL_SECONDS,
        "n": secrets.token_urlsafe(8),
    }
    raw = base64.urlsafe_b64encode(json.dumps(payload).encode()).decode()
    sig = hmac.new(
        settings.FACEBOOK_APP_SECRET.encode(), raw.encode(), hashlib.sha256
    ).hexdigest()
    return f"{raw}.{sig}"


def _parse_state(state: str) -> tuple[str, str]:
    try:
        raw, sig = state.rsplit(".", 1)
    except ValueError:
        raise HTTPException(status_code=400, detail="Malformed state")
    expected = hmac.new(
        settings.FACEBOOK_APP_SECRET.encode(), raw.encode(), hashlib.sha256
    ).hexdigest()
    if not hmac.compare_digest(sig, expected):
        raise HTTPException(status_code=400, detail="Invalid state signature")
    payload = json.loads(base64.urlsafe_b64decode(raw.encode()).decode())
    if payload["exp"] < int(time.time()):
        raise HTTPException(status_code=400, detail="State expired")
    return payload["slug"], payload["agency_id"]


def _resolve_slug(request: Request) -> str:
    slug: Optional[str] = getattr(request.state, "tenant_slug", None)
    if not slug:
        raise HTTPException(status_code=400, detail="No tenant in request")
    return slug


def _redirect_uri() -> str:
    return f"{settings.PUBLIC_BASE_URL}/api/auth/meta/callback"


def _dialog_url(state: str, scopes: list[str]) -> str:
    from urllib.parse import urlencode

    query = urlencode(
        {
            "client_id": settings.FACEBOOK_APP_ID,
            "redirect_uri": _redirect_uri(),
            "state": state,
            "scope": ",".join(scopes),
            "response_type": "code",
        }
    )
    return f"https://www.facebook.com/{settings.META_GRAPH_VERSION}/dialog/oauth?{query}"


def _close_window_html(ok: bool, message: str) -> HTMLResponse:
    colour = "#16a34a" if ok else "#dc2626"
    return HTMLResponse(
        content=(
            "<html><body style='font-family:system-ui;text-align:center;padding:60px'>"
            f"<h2 style='color:{colour}'>{message}</h2>"
            "<p style='color:#6b7280;font-size:14px'>You can close this window.</p>"
            f"<script>window.opener && window.opener.postMessage('{META_OAUTH_COMPLETE}','*')</script>"
            "</body></html>"
        )
    )


@auth_router.get("/connect")
async def connect_facebook(
    request: Request, _user: User = Depends(require_permission("settings:manage"))
) -> dict[str, str]:
    if not settings.FACEBOOK_APP_ID:
        raise HTTPException(status_code=400, detail="Meta is not configured")
    state = _make_state(_resolve_slug(request), DEFAULT_AGENCY_ID)
    return {"authorize_url": _dialog_url(state, FACEBOOK_SCOPES)}


@auth_router.get("/connect/instagram")
async def connect_instagram(
    request: Request, _user: User = Depends(require_permission("settings:manage"))
) -> dict[str, str]:
    if not settings.FACEBOOK_APP_ID:
        raise HTTPException(status_code=400, detail="Meta is not configured")
    slug = _resolve_slug(request)
    factory = get_tenant_session_factory(slug)
    async with factory() as session:
        existing = await session.scalar(
            select(AgencySocialConnection.id).where(
                AgencySocialConnection.agency_id == DEFAULT_AGENCY_ID,
                AgencySocialConnection.platform == "facebook",
            )
        )
    if not existing:
        raise HTTPException(
            status_code=409, detail="Connect a Facebook Page first"
        )
    state = _make_state(slug, DEFAULT_AGENCY_ID)
    return {"authorize_url": _dialog_url(state, FACEBOOK_SCOPES + INSTAGRAM_SCOPES)}


@auth_router.get("/callback")
async def meta_callback(
    code: str | None = None,
    state: str | None = None,
    error: str | None = None,
) -> HTMLResponse:
    if error or not code or not state:
        return _close_window_html(False, "Meta connection cancelled")
    slug, agency_id = _parse_state(state)
    client = MetaGraphClient()
    factory = get_tenant_session_factory(slug)
    try:
        short = await client.exchange_code(code, _redirect_uri())
        long_lived = await client.exchange_for_long_lived(short)
        user_token = long_lived["access_token"]
        expires_at = datetime.now(timezone.utc).timestamp() + int(
            long_lived.get("expires_in", 0)
        )
        token_expires_at = datetime.fromtimestamp(expires_at, tz=timezone.utc)
        pages = await client.get_pages(user_token)
    except MetaGraphError:
        logger.exception("meta_callback.exchange_failed", slug=slug)
        return _close_window_html(False, "Could not connect to Meta")

    async with factory() as session:
        await _store_connection(
            session, agency_id, user_token, token_expires_at, pages, client
        )
        await session.commit()
    return _close_window_html(True, "Facebook connected successfully")


async def _store_connection(
    session: AsyncSession,
    agency_id: str,
    user_token: str,
    token_expires_at: datetime,
    pages: list[dict],
    client: MetaGraphClient,
) -> None:
    now = datetime.now(timezone.utc)
    await session.execute(
        delete(AgencySocialConnection).where(
            AgencySocialConnection.agency_id == agency_id
        )
    )
    first = True
    for page in pages:
        session.add(
            AgencySocialConnection(
                agency_id=agency_id,
                platform="facebook",
                page_id=page["id"],
                page_name=page.get("name"),
                access_token=_obfuscate(page["access_token"]),
                is_primary=first,
                user_access_token=_obfuscate(user_token),
                token_expires_at=token_expires_at,
                warning_level=0,
                status="connected",
                connected_at=now,
            )
        )
        ig = await client.get_ig_account(page["id"], page["access_token"])
        if ig:
            session.add(
                AgencySocialConnection(
                    agency_id=agency_id,
                    platform="instagram",
                    ig_account_id=ig["id"],
                    ig_username=ig.get("username"),
                    access_token=_obfuscate(page["access_token"]),
                    is_primary=False,
                    user_access_token=_obfuscate(user_token),
                    token_expires_at=token_expires_at,
                    warning_level=0,
                    status="connected",
                    connected_at=now,
                )
            )
        first = False


@data_router.get("/connections")
async def list_connections(request: Request) -> list[dict]:
    slug = _resolve_slug(request)
    factory = get_tenant_session_factory(slug)
    async with factory() as session:
        result = await session.execute(
            select(AgencySocialConnection).where(
                AgencySocialConnection.agency_id == DEFAULT_AGENCY_ID
            )
        )
        rows = result.scalars().all()
    return [
        {
            "id": str(r.id),
            "platform": r.platform,
            "page_id": r.page_id,
            "ig_account_id": r.ig_account_id,
            "page_name": r.page_name,
            "ig_username": r.ig_username,
            "is_primary": r.is_primary,
            "status": r.status,
            "token_expires_at": (
                r.token_expires_at.isoformat() if r.token_expires_at else None
            ),
            "warning_level": r.warning_level,
        }
        for r in rows
    ]


@data_router.delete("/connections", status_code=204)
async def disconnect(request: Request) -> None:
    slug = _resolve_slug(request)
    factory = get_tenant_session_factory(slug)
    async with factory() as session:
        await session.execute(
            delete(AgencySocialConnection).where(
                AgencySocialConnection.agency_id == DEFAULT_AGENCY_ID
            )
        )
        await session.commit()
```

- [ ] **Step 2: Register both routers in `src/main.py`**

Add imports near the other router imports at the top of `src/main.py`:

```python
from src.api.meta_social import auth_router as meta_auth_router
from src.api.meta_social import data_router as meta_data_router
```

In `create_app()`, after the `application.include_router(trademe_oauth_router)` line, add:

```python
    application.include_router(meta_auth_router)
    application.include_router(meta_data_router)
```

- [ ] **Step 3: Verify the app boots and routes are registered**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.main import app; print([r.path for r in app.routes if '/meta' in getattr(r, 'path', '')])"`
Expected: includes `/api/auth/meta/connect`, `/api/auth/meta/connect/instagram`, `/api/auth/meta/callback`, `/api/social/meta/connection`.

- [ ] **Step 4: Commit** — SKIP (execution override: leave changes in the working tree).

---

### Task 5b: Encrypt tokens at rest with Fernet (security hardening)

**Why:** Tokens are currently **base64-obfuscated, not encrypted** — `src/utils.py`'s
`obfuscate`/`deobfuscate` are plain `base64.b64encode/decode` (no key). Anyone with tenant-DB
read access, a backup, or a logged query can recover live Page/User tokens in one line.
Replace the obfuscation with real authenticated encryption (Fernet = AES-128-CBC + HMAC).
`cryptography==46.0.5` is **already** in `requirements.txt` — no new dependency.

**Scope:** `src/utils.py`'s `obfuscate`/`deobfuscate` are used **only** by the Meta feature
(`meta_social_service.py`, and the monitor/tests in Tasks 10/13). Every other credential
store (WhatsApp, Telegram, SMS, platform creds, LLM) has its **own** private base64
`_obfuscate`/`_deobfuscate` and is **out of scope** here — they share the same weakness and
can adopt this helper later, but that's a separate change.

**Files:**

- Modify: `src/config.py`
- Modify: `src/utils.py`
- Modify: `src/services/meta_social_service.py`
- Create: `scripts/generate_token_encryption_key.py`
- Modify (deploy rollout): `.env.template`, `.env.example`, slicify-control `src/config.py` + `src/services/provisioner.py` + `.env.example`
- Dev only: `.env.local`

- [ ] **Step 1: Add the key to config**

In `src/config.py`, add alongside the other Meta/secret settings:

```python
    TOKEN_ENCRYPTION_KEY: str = ""
```

Empty default keeps non-Meta code unaffected; encrypt/decrypt fail loudly if it's unset when
the Meta feature is actually used (Step 2).

- [ ] **Step 2: Replace obfuscation with Fernet in `src/utils.py`**

Replace the two base64 functions:

```python
def obfuscate(value: str) -> str:
    return base64.b64encode(value.encode()).decode()


def deobfuscate(encoded: str) -> str:
    return base64.b64decode(encoded.encode()).decode()
```

with key-backed encrypt/decrypt (rename to honest names):

```python
from cryptography.fernet import Fernet

from src.config import settings


def _fernet() -> Fernet:
    if not settings.TOKEN_ENCRYPTION_KEY:
        raise RuntimeError("TOKEN_ENCRYPTION_KEY is not configured")
    return Fernet(settings.TOKEN_ENCRYPTION_KEY.encode())


def encrypt(value: str) -> str:
    return _fernet().encrypt(value.encode()).decode()


def decrypt(token: str) -> str:
    return _fernet().decrypt(token.encode()).decode()
```

Put the `Fernet`/`settings` imports at the top with the other imports (order: stdlib →
third-party → local). `src/config.py` does **not** import `src/utils.py`, so there's no
circular import. Remove `import base64` if it's now unused in the file (it is, unless other
code references it — verify with a grep).

- [ ] **Step 3: Update the Meta call sites**

In `src/services/meta_social_service.py`, change `from src.utils import obfuscate` to
`from src.utils import encrypt`, and replace the three `obfuscate(...)` calls
(`user_access_token=`, the two account `access_token=`) with `encrypt(...)`.

> The monitor (Task 10) and its tests (Task 13) — re-synced to the two-table model — import
> `decrypt`/`encrypt` from `src.utils` (not `deobfuscate`/`obfuscate`). Use the new names
> when those tasks are written.

- [ ] **Step 4: Dev key + clear stale base64 rows**

Generate a dev key straight into `.env.local` (the file the local Docker backend mounts as
`/app/.env`) with the helper script — `--write` fills/appends the line, `--force` rotates:

```bash
docker compose -f docker-compose.local.yml run --rm --no-deps backend \
  python scripts/generate_token_encryption_key.py --write .env.local
```

(Bare `python scripts/generate_token_encryption_key.py` just prints a key; `--env-line`
prints a paste-ready `TOKEN_ENCRYPTION_KEY=<key>` line.) Existing dev rows hold **base64**,
which Fernet can't decrypt — clear them (tokens are re-derivable by reconnecting):

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c "DELETE FROM agency_social_connections;"
```

Restart the backend, then reconnect via the Meta tab so tokens are re-stored **encrypted**.

- [ ] **Step 5: Verify**

Round-trip + prove the stored value is no longer plain base64:

```bash
docker compose -f docker-compose.local.yml run --rm --no-deps backend python -c "
import base64
from src.utils import encrypt, decrypt
c1, c2 = encrypt('EAAsecret'), encrypt('EAAsecret')
assert decrypt(c1) == 'EAAsecret'
assert c1 != c2  # Fernet uses a random IV per call
try:
    print('base64-decodes to token?', base64.b64decode(c1).decode(errors='ignore').startswith('EAA'))
except Exception as e:
    print('not plain base64 ->', type(e).__name__)
print('ok')
"
```

Expected: `ok`, and the value does **not** base64-decode to an `EAA…` token. After Step 4's
reconnect, re-run the DB probe from earlier — the stored column should no longer decode to a
live token.

- [ ] **Step 6: Deploy rollout (shared env key — NOT run during local dev)**

Mirror the shared-key flow in `CLAUDE.md` ("Rolling out a new shared env var to every tenant"):

1. `.env.template` — add `TOKEN_ENCRYPTION_KEY={{TOKEN_ENCRYPTION_KEY}}`.
2. `.env.example` — add a documented `TOKEN_ENCRYPTION_KEY=` line.
3. slicify-control — add `slate_token_encryption_key` to `src/config.py`, a substitution in
   `src/services/provisioner.py`'s `replacements`, and a line in its `.env.example`.
4. Patch existing tenants (one shared key across tenants; dry-run first):

   ```bash
   venv/bin/python scripts/patch_tenant_envs.py --set TOKEN_ENCRYPTION_KEY=<key> \
       --section "Meta token encryption" --dry-run
   ```

5. Restart slate.

> **Ordering caveat:** this must ship **before** any tenant connects Meta in production —
> once live, switching the cipher would orphan already-stored tokens (Fernet can't read old
> base64). No prod tokens exist yet, so there's nothing to migrate; UAT/dev just
> disconnect+reconnect. A single app-wide key is used (simplest, matches the other shared
> secrets); per-tenant keys are a possible later hardening. Losing the key means every stored
> token must be re-fetched via reconnect — it is **not** otherwise recoverable.

- [ ] **Step 7: Commit** — SKIP (execution override: leave changes in the working tree).

---

### Task 6: Frontend types + API client

**Files:**
- Create: `frontend/src/types/meta.ts`
- Create: `frontend/src/api/metaCredentials.ts`

> **As-built:** the codebase keeps one type file per domain (`src/types/whatsapp.ts`, `platform.ts`, …) imported directly, rather than one growing `index.ts`. So these interfaces live in a dedicated **`frontend/src/types/meta.ts`** and are imported as `from "../types/meta"` (not `from "../types"`). `MetaSettingsPage.tsx` (Task 7) imports `MetaConnectResponse` from `../types/meta` as well.

- [ ] **Step 1: Add types**

Create `frontend/src/types/meta.ts` (two-table shape — one connection per agency, nesting its accounts):

```ts
export type MetaPlatform = "facebook" | "instagram";

export interface MetaAccount {
  id: string;
  platform: MetaPlatform;
  provider_id: string;
  name: string | null;
  username: string | null;
  is_primary: boolean;
  authorized: boolean;
}

export interface MetaConnection {
  id: string;
  status: string;
  token_expires_at: string | null;
  warning_level: number;
  connected_at: string | null;
  accounts: MetaAccount[];
}

export interface MetaConnectResponse {
  authorize_url: string;
}
```

- [ ] **Step 2: Add API calls**

Create `frontend/src/api/metaCredentials.ts` (the GET returns the single connection, or `null` when none):

```ts
import { del, get } from "./client";
import type { MetaConnection, MetaConnectResponse } from "../types/meta";

export const fetchMetaConnection = () =>
  get<MetaConnection | null>("/social/meta/connection");

export const metaConnectFacebook = () =>
  get<MetaConnectResponse>("/auth/meta/connect");

export const metaConnectInstagram = () =>
  get<MetaConnectResponse>("/auth/meta/connect/instagram");

export const metaDisconnect = () => del("/social/meta/connection");
```

- [ ] **Step 3: Verify the frontend type-checks**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit`
Expected: no errors referencing `metaCredentials.ts` or `MetaConnection`.

- [ ] **Step 4: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 7: Frontend — Meta tab panel + card

**Files:**
- Create: `frontend/src/components/integrations/MetaAccountCard.tsx`
- Create: `frontend/src/pages/MetaSettingsPage.tsx`
- Modify: `frontend/src/pages/IntegrationsPage.tsx`

- [ ] **Step 1: Presentational card**

Create `frontend/src/components/integrations/MetaAccountCard.tsx`:

```tsx
interface MetaAccountCardProps {
  title: string;
  subtitle: string;
  iconBg: string;
  iconChar: string;
  connected: boolean;
  disabled: boolean;
  buttonLabel: string;
  onConnect: () => void;
}

const PILL_CONNECTED = "bg-green-50 text-green-700";
const PILL_NOT_CONNECTED = "bg-gray-100 text-gray-600";
const DOT_CONNECTED = "bg-green-500";
const DOT_NOT_CONNECTED = "bg-gray-400";

export function MetaAccountCard(props: MetaAccountCardProps) {
  const pill = props.connected ? PILL_CONNECTED : PILL_NOT_CONNECTED;
  const dot = props.connected ? DOT_CONNECTED : DOT_NOT_CONNECTED;
  const pillLabel = props.connected ? "Connected" : "Not connected";
  return (
    <div className="rounded-xl border border-gray-200 bg-white p-6 shadow-sm">
      <div className="flex items-center justify-between">
        <div className="flex items-center gap-4">
          <div
            className={`flex h-10 w-10 items-center justify-center rounded-lg text-lg font-bold text-white ${props.iconBg}`}
          >
            {props.iconChar}
          </div>
          <div>
            <p className="font-semibold text-gray-900">{props.title}</p>
            <p className="text-sm text-gray-500">{props.subtitle}</p>
          </div>
        </div>
        <span
          className={`inline-flex items-center gap-1.5 rounded-full px-2.5 py-1 text-xs font-medium ${pill}`}
        >
          <span className={`h-1.5 w-1.5 rounded-full ${dot}`} />
          {pillLabel}
        </span>
      </div>
      <button
        type="button"
        onClick={props.onConnect}
        disabled={props.disabled}
        className="mt-4 rounded-lg bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700 disabled:cursor-not-allowed disabled:opacity-50"
      >
        {props.buttonLabel}
      </button>
    </div>
  );
}
```

> **As-built (connected-state button):** `MetaAccountCard` derives its button from the `connected` prop — when connected it renders a secondary-styled **"Reconnect"** button (re-runs the same idempotent connect flow); when not connected it renders the primary `buttonLabel`. Label/style constants (`RECONNECT_LABEL`, `BUTTON_BASE`/`BUTTON_PRIMARY`/`BUTTON_SECONDARY`) live at the top of the card file.

- [ ] **Step 2: Panel that fetches + orchestrates**

Create `frontend/src/pages/MetaSettingsPage.tsx`:

```tsx
import { useEffect } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import {
  fetchMetaConnections,
  metaConnectFacebook,
  metaConnectInstagram,
} from "../api/metaCredentials";
import type { MetaConnectResponse } from "../types";
import { MetaAccountCard } from "../components/integrations/MetaAccountCard";

const META_OAUTH_COMPLETE = "meta-oauth-complete";
const POPUP_FEATURES = "width=600,height=700";
const CONNECTIONS_KEY = ["meta-connections"];

function openOAuthPopup(data: MetaConnectResponse) {
  window.open(data.authorize_url, "meta-oauth", POPUP_FEATURES);
}

export function MetaSettingsPanel() {
  const queryClient = useQueryClient();
  const { data: connections } = useQuery({
    queryKey: CONNECTIONS_KEY,
    queryFn: fetchMetaConnections,
  });

  const facebookMutation = useMutation({
    mutationFn: metaConnectFacebook,
    onSuccess: openOAuthPopup,
  });
  const instagramMutation = useMutation({
    mutationFn: metaConnectInstagram,
    onSuccess: openOAuthPopup,
  });

  useEffect(() => {
    function handler(event: MessageEvent) {
      if (event.data === META_OAUTH_COMPLETE) {
        queryClient.invalidateQueries({ queryKey: CONNECTIONS_KEY });
      }
    }
    window.addEventListener("message", handler);
    return () => window.removeEventListener("message", handler);
  }, [queryClient]);

  const facebookConnected = (connections ?? []).some(
    (c) => c.platform === "facebook",
  );
  const instagramConnected = (connections ?? []).some(
    (c) => c.platform === "instagram",
  );

  function handleConnectFacebook() {
    facebookMutation.mutate();
  }
  function handleConnectInstagram() {
    instagramMutation.mutate();
  }

  return (
    <div className="space-y-4">
      <div>
        <h3 className="font-semibold text-gray-900">Connected accounts</h3>
        <p className="text-sm text-gray-500">
          Connect your Facebook Page first — Instagram requires a linked
          Facebook Page.
        </p>
      </div>
      <MetaAccountCard
        title="Facebook"
        subtitle="Publish posts to your Facebook Page"
        iconBg="bg-blue-600"
        iconChar="f"
        connected={facebookConnected}
        disabled={false}
        buttonLabel="Connect Facebook"
        onConnect={handleConnectFacebook}
      />
      <MetaAccountCard
        title="Instagram"
        subtitle="Connect your Facebook Page first"
        iconBg="bg-pink-500"
        iconChar="IG"
        connected={instagramConnected}
        disabled={!facebookConnected}
        buttonLabel="Connect Instagram"
        onConnect={handleConnectInstagram}
      />
    </div>
  );
}
```

> **As-built (two-table data shape):** the panel calls `fetchMetaConnection()` (singular → `MetaConnection | null`, query key `["meta-connection"]`) instead of a list. Connection state is derived from the nested accounts: `const accounts = connection?.accounts ?? []; facebookConnected = accounts.some(a => a.platform === PLATFORM_FACEBOOK); instagramConnected = accounts.some(a => a.platform === PLATFORM_INSTAGRAM)` (platform literals are file-scoped constants). The OAuth-complete message handler invalidates `["meta-connection"]`.
>
> **As-built fix (popup blocker):** opening `window.open(authorize_url, …)` inside the mutation's `onSuccess` is blocked by browsers because it runs *after* the awaited request, outside the click's user-gesture. The shipped `MetaSettingsPage.tsx` instead opens a blank popup **synchronously** in the click handler, then sets `popup.location.href` once the URL returns (and `popup.close()` on error):
>
> ```tsx
> const POPUP_NAME = "meta-oauth";
> function openBlankPopup() {
>   return window.open("", POPUP_NAME, POPUP_FEATURES);
> }
> function navigatePopup(popup: Window | null, data: MetaConnectResponse) {
>   if (!popup) {
>     return;
>   }
>   popup.location.href = data.authorize_url;
> }
> // mutations are plain useMutation({ mutationFn }); handlers do:
> function handleConnectFacebook() {
>   const popup = openBlankPopup();
>   facebookMutation.mutate(undefined, {
>     onSuccess: (data) => navigatePopup(popup, data),
>     onError: () => popup?.close(),
>   });
> }
> ```

- [ ] **Step 3: Add the Meta tab to the Integrations page**

In `frontend/src/pages/IntegrationsPage.tsx`, add to the `FIXED_TABS` array:

```tsx
  { key: "meta", label: "Meta" },
```

Add the import near the other panel imports:

```tsx
import { MetaSettingsPanel } from "./MetaSettingsPage";
```

Add the dispatch line alongside the other `isFixedTab && activeTab === ...` lines:

```tsx
{isFixedTab && activeTab === "meta" && <MetaSettingsPanel />}
```

- [ ] **Step 4: Verify type-check and build**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit && npm run build`
Expected: no type errors; build succeeds.

- [ ] **Step 5: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 8: Manual end-to-end verification (Milestone 1)

**Pre-req — environment config.** The local backend runs in **Docker** (`docker-compose.local.yml`), and that container mounts **`.env.local`** as `/app/.env` — **not** the repo's `.env`. So the Meta keys must be in `.env.local` (the `scripts/patch_tenant_envs.py` / `.env.template` path is for tenant `.env_<slug>` files, separate concern). Set in `.env.local`:

```
FACEBOOK_APP_ID=...
FACEBOOK_APP_SECRET=...
PUBLIC_BASE_URL=https://local.slatedev.com
# META_GRAPH_VERSION defaults to v21.0 in src/config.py — only set to override
```

After editing `.env.local`, restart the backend so config reloads:
`docker compose -f docker-compose.local.yml restart backend`.

**Pre-req — Meta app dashboard.** A real Meta app must exist, and **three different fields** must be configured (each rejects a different format — domain vs full URL):

| Dashboard field                  | Location                  | Value                                                          |
| -------------------------------- | ------------------------- | -------------------------------------------------------------- |
| **App Domains**                  | Settings → Basic          | `local.slatedev.com` (bare domain)                             |
| **Valid OAuth Redirect URIs**    | Facebook Login → Settings | `https://local.slatedev.com/api/auth/meta/callback` (full URL) |
| **Allowed Domains for redirect** | Facebook Login → Settings | `local.slatedev.com` (bare domain)                             |

Use the **Redirect URI Validator** on the Facebook Login → Settings page to confirm the full callback URL turns green before testing.

**Pre-req — app mode / scopes.** Requested scopes are `pages_show_list`, `pages_read_engagement`, `business_management` (+ `instagram_basic` for IG). `pages_manage_metadata` was removed — Meta returns `Invalid Scopes` for it on this app type. These are **advanced** permissions: in **Development mode** they're granted only to accounts with an app **role** (Admin/Developer/Tester); production needs **App Review**. `redirect_uri` must be the **absolute** `{PUBLIC_BASE_URL}/...` — an empty `PUBLIC_BASE_URL` yields a relative `/api/...` and a blank Facebook dialog.

- [ ] **Step 1: Run backend + frontend**

Backend already runs as the `slicify-realestate-backend` Docker container (uvicorn `--reload`, source mounted at `/app`) behind Caddy on `https://local.slatedev.com`. Confirm it's up: `docker compose -f docker-compose.local.yml ps`. Start the frontend: `cd frontend && npm run dev`.

- [ ] **Step 2: Connect Facebook**

Navigate to Integrations → Meta. Click **Connect Facebook**. Complete the Facebook OAuth dialog in the popup. The popup should show "Facebook connected successfully" and close; the Facebook card pill flips to **Connected**.

- [ ] **Step 3: Verify rows landed**

Run (local Postgres runs in Docker): `docker compose -f docker-compose.local.yml exec -T postgres psql -U slicify -d sideline_realestate -c "SELECT platform, page_name, ig_username, is_primary, status, token_expires_at IS NOT NULL AS has_expiry FROM agency_social_connections;"`
Expected: one `facebook` row per Page (`is_primary=t` on the first), plus an `instagram` row for any Page with a linked IG Business account; `status=connected`, `has_expiry=t`.

> **Verified (2026-06-05):** Facebook path confirmed end-to-end — Page **"Slicify Test"** stored (`is_primary=t`, `status=connected`, expiry set). No IG row because that Page has no linked IG Business account (expected). IG path still needs a Page with a linked IG Business account to exercise fully.

- [ ] **Step 4: Verify disconnect / reconnect**

There is no Disconnect button in the UI (chosen behaviour: a **Reconnect** button on the connected card re-runs the idempotent connect flow). To clear rows, call the API directly (`curl -X DELETE` with auth, or `DELETE /api/social/meta/connections`) and re-run the query → zero rows. Then **Reconnect** from the card and confirm no duplicate rows.

---

## Milestone 2 — Instagram after Facebook

The backend already auto-derives IG during the Facebook connect (Task 5 `MetaSocialService._store` / `_store_page`) and exposes `/api/auth/meta/connect/instagram` (gated on an existing Facebook row). The frontend already disables the Instagram card until Facebook is connected (Task 7). This milestone is verification + the explicit IG-scope re-consent path.

### Task 9: Manual verification of Instagram gating + connect

- [ ] **Step 1: Verify the gate**

With no Facebook connection (after a disconnect), confirm the **Connect Instagram** button is greyed out / disabled and `GET /api/auth/meta/connect/instagram` returns `409 Connect a Facebook Page first`.

Run (with a valid bearer token): `curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" "$PUBLIC_BASE_URL/api/auth/meta/connect/instagram"`
Expected: `409`.

- [ ] **Step 2: Verify connect-after-Facebook**

Connect Facebook first, then click **Connect Instagram**. The popup runs the OAuth with IG scopes and returns success. Confirm the `instagram` row(s) exist and the Instagram pill shows **Connected**:

Run: `docker compose -f docker-compose.local.yml exec -T postgres psql -U slicify -d sideline_realestate -c "SELECT platform, ig_username, status FROM agency_social_connections WHERE platform='instagram';"`
Expected: one row per linked IG Business account, `status=connected`.

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

## Milestone 3 — Daily token monitor + silent refresh + warnings (with tests)

> **Two-table, connection-level (re-synced).** The monitor iterates **one
> `AgencySocialConnection` per agency** (not per-Page rows) and writes the lifecycle fields
> (`status`, `warning_level`, `token_expires_at`, `user_access_token`, `last_refreshed_at`,
> `last_warned_at`, `last_error`) directly onto that single connection row — no more "UPDATE
> all agency rows". Tests seed a connection + a facebook `AgencySocialAccount` child.
>
> **Refresh strategy (resolved — keep silent refresh).** When ≤14 days remain the loop calls
> `refresh_long_lived` (`fb_exchange_token`) on the user token and, on success, stores the
> returned token + expiry; a genuine extension past 14 days clears prior warnings. On a hard
> failure (expired/revoked) it falls through to `needs_reconnect` + a one-time reconnect
> notification. **Honest caveat:** for server-side apps Meta often returns a token with
> roughly the _same_ remaining lifetime and `fb_exchange_token` does **not** reset the 60-day
> clock, and Page tokens don't expire while the user token is valid — so silent refresh is
> best-effort and the 14/5-day warning + reconnect path is the guaranteed backstop. Page
> tokens are **not** re-fetched on refresh (they stay valid).

### Task 10: `meta_token_monitor` service

**Files:**
- Modify: `src/db/repositories/agency_social_connection.py`
- Create: `src/services/meta_token_monitor.py`

> **Refactor-aligned:** keeps the functional `process_agency_tokens(session, client, now)` API (client injected as a parameter — same stateless pattern as `runtime_settings.py`, and what the tests already call), but all DB access goes through `AgencySocialConnectionRepository` / `UserRepository` (no inline `select`/`update`/`session.get`), and statuses use the `MetaConnectionStatus` enum.

> **As-built (re-synced — AUTHORITATIVE; supersedes the "Refactor-aligned" note and the Step 1/Step 2 code blocks below):**
>
> - **Class-based service, mirroring `MetaSocialService`.** `class MetaTokenMonitor(session, graph_client=None)` injects `self.session`, `self.connections = AgencySocialConnectionRepository(session)`, and `self.graph = graph_client or MetaGraphClient()` (so tests pass a fake). Logic lives in instance methods: `process_agency_tokens(now)`, `_process_one_agency`, `_refresh_user_token`, `_apply_status`. The loop/driver stays **module-level** (`_run_for_factory`/`_run_default`/`_run_tenant`/`meta_token_monitor_loop`): it owns the session/engine lifecycle and instantiates `MetaTokenMonitor(db, client)` per database (default DB + every tenant), exactly as `MetaSocialService` keeps its stateless OAuth helpers at module level.
> - **Listing reuses the inherited `BaseRepository.get_all()` — NOT `list_primary()`.** The plan's `list_primary()` filters `AgencySocialConnection.is_primary`, but in the two-table model `is_primary` is a column on `agency_social_accounts`, **not** the connection. There is one connection per agency, so the monitor just iterates `get_all()`. The **only** new repo method is `update_by_agency(agency_id, **values)` (add `update` to the sqlalchemy import). Drop the `list_primary()` block in Step 1.
> - **`encrypt`/`decrypt`, not `obfuscate`/`deobfuscate`** (Task 5b): `decrypt(connection.user_access_token)` on refresh, `encrypt(refreshed["access_token"])` on store.
> - **No `notify_reconnect` — in-app notification only.** The refresh-failure branch persists the in-app signal and returns:
>   ```python
>   except MetaGraphError:
>       await self.connections.update_by_agency(
>           connection.agency_id,
>           status=MetaConnectionStatus.NEEDS_RECONNECT.value,
>           last_error="silent refresh failed",
>       )
>       return None
>   ```
>   `status` is exposed by `MetaConnectionResponse.status` and surfaced in the UI by `frontend/src/components/integrations/MetaStatusBanner.tsx` (red for `needs_reconnect`/`expired`/`error`, amber for `expiring`). This removes the email path entirely — **no `communications` agent, no `send_notification`, and `get_admins` (Task 11) is no longer needed.**
> - **Verify (Step 2):** `from src.services.meta_token_monitor import MetaTokenMonitor, meta_token_monitor_loop`; exercise via `await MetaTokenMonitor(session, fake_client).process_agency_tokens(now)`.

- [ ] **Step 1: Add the repository methods**

Add to `AgencySocialConnectionRepository` in `src/db/repositories/agency_social_connection.py` (the file from Task 5's refactor):

```python
    async def list_primary(self) -> list[AgencySocialConnection]:
        stmt = select(AgencySocialConnection).where(
            AgencySocialConnection.is_primary.is_(True)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def update_by_agency(self, agency_id: str, **values) -> None:
        await self.session.execute(
            update(AgencySocialConnection)
            .where(AgencySocialConnection.agency_id == agency_id)
            .values(**values)
        )
```

Add `update` to the existing sqlalchemy import: `from sqlalchemy import delete, select, update`.

- [ ] **Step 2: Write the service**

Create `src/services/meta_token_monitor.py`:

```python
import asyncio
from datetime import datetime, timedelta, timezone

import asyncpg
import structlog
from sqlalchemy.ext.asyncio import AsyncSession

from src.config import list_available_tenants
from src.constants import MetaConnectionStatus
from src.db.repositories.agency_social_connection import (
    AgencySocialConnectionRepository,
)
from src.db.repositories.user import UserRepository
from src.db.session import async_session_factory, get_tenant_session_factory
from src.integrations.meta_graph import MetaGraphClient, MetaGraphError
from src.models.agency_social_connection import AgencySocialConnection
from src.utils import deobfuscate, obfuscate

logger = structlog.get_logger()

CHECK_INTERVAL_SECONDS = 24 * 60 * 60
WARN_NONE = 0
WARN_14 = 14
WARN_5 = 5
RECONNECT_MESSAGE = (
    "Your Meta (Facebook/Instagram) connection in Slate could not be "
    "refreshed automatically and needs to be reconnected. Open "
    "Integrations -> Meta and click Connect Facebook again."
)


async def notify_reconnect(
    session: AsyncSession, agency_id: str, connected_by_user_id
) -> None:
    from src.internal_agents import get_internal_agent

    users = UserRepository(session)
    emails: list[str] = []
    if connected_by_user_id:
        initiator = await users.get_by_id(connected_by_user_id)
        if initiator and initiator.is_active:
            emails = [initiator.email]
    if not emails:
        emails = [u.email for u in await users.get_admins()]
    if not emails:
        logger.warning("meta_monitor.no_recipient", agency_id=agency_id)
        return
    comms = get_internal_agent("communications")
    for email in emails:
        await comms.run_skill(
            "send_notification",
            session,
            channels=["email"],
            recipient=email,
            message=RECONNECT_MESSAGE,
        )


async def _process_one_agency(
    session: AsyncSession,
    repo: AgencySocialConnectionRepository,
    client: MetaGraphClient,
    primary: AgencySocialConnection,
    now: datetime,
) -> None:
    if not primary.user_access_token or not primary.token_expires_at:
        return
    agency_id = primary.agency_id
    token_expires_at = primary.token_expires_at
    days_remaining = (token_expires_at - now).days

    if days_remaining <= WARN_14:
        try:
            refreshed = await client.refresh_long_lived(
                deobfuscate(primary.user_access_token)
            )
        except MetaGraphError:
            await repo.update_by_agency(
                agency_id,
                status=MetaConnectionStatus.NEEDS_RECONNECT.value,
                last_error="silent refresh failed",
            )
            await notify_reconnect(session, agency_id, primary.connected_by_user_id)
            return
        new_expiry = now + timedelta(seconds=int(refreshed["expires_in"]))
        await repo.update_by_agency(
            agency_id,
            user_access_token=obfuscate(refreshed["access_token"]),
            token_expires_at=new_expiry,
            last_refreshed_at=now,
        )
        token_expires_at = new_expiry
        days_remaining = (token_expires_at - now).days

    if days_remaining <= 0:
        await repo.update_by_agency(
            agency_id, status=MetaConnectionStatus.EXPIRED.value
        )
        return
    if days_remaining <= WARN_5 and primary.warning_level != WARN_5:
        await repo.update_by_agency(
            agency_id,
            warning_level=WARN_5,
            status=MetaConnectionStatus.EXPIRING.value,
            last_warned_at=now,
        )
        return
    if days_remaining <= WARN_14 and primary.warning_level == WARN_NONE:
        await repo.update_by_agency(
            agency_id,
            warning_level=WARN_14,
            status=MetaConnectionStatus.EXPIRING.value,
            last_warned_at=now,
        )
        return
    if days_remaining > WARN_14:
        await repo.update_by_agency(
            agency_id,
            warning_level=WARN_NONE,
            status=MetaConnectionStatus.CONNECTED.value,
        )


async def process_agency_tokens(
    session: AsyncSession, client: MetaGraphClient, now: datetime
) -> None:
    repo = AgencySocialConnectionRepository(session)
    for primary in await repo.list_primary():
        await _process_one_agency(session, repo, client, primary, now)
    await session.flush()


async def meta_token_monitor_loop() -> None:
    logger.info("meta_monitor.started", interval=CHECK_INTERVAL_SECONDS)
    while True:
        client = MetaGraphClient()
        now = datetime.now(timezone.utc)
        try:
            async with async_session_factory() as db:
                await process_agency_tokens(db, client, now)
                await db.commit()
        except Exception:
            logger.exception("meta_monitor.default_error")
        for slug in list_available_tenants():
            try:
                factory = get_tenant_session_factory(slug)
                async with factory() as db:
                    await process_agency_tokens(db, client, now)
                    await db.commit()
            except asyncpg.exceptions.InvalidCatalogNameError:
                logger.warning("meta_monitor.tenant_db_missing", tenant=slug)
            except Exception:
                logger.exception("meta_monitor.tenant_error", tenant=slug)
        await asyncio.sleep(CHECK_INTERVAL_SECONDS)
```

- [ ] **Step 2: Verify it imports**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/python -c "from src.services.meta_token_monitor import MetaTokenMonitor, meta_token_monitor_loop; print('ok')"`
Expected: `ok`

- [ ] **Step 4: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 11: Add `get_admins()` to `UserRepository`

> **As-built — NOT required for this feature (skipped).** `get_admins()` existed only to pick admin-fallback recipients for `notify_reconnect`. The monitor no longer sends email — reconnect is surfaced in-app (Task 10 as-built) — so nothing calls `get_admins()`. Leave this task unimplemented unless a future feature needs it; the snippet below still stands if/when it does.

**Files:**
- Modify: `src/db/repositories/user.py`

- [ ] **Step 1: Add the method**

Add to the `UserRepository` class. Add `from src.constants import UserRole` (and ensure `from src.models.user import User` and `from sqlalchemy import select` are imported):

```python
    async def get_admins(self) -> list[User]:
        result = await self.session.execute(
            select(User).where(
                User.role.in_([UserRole.OWNER.value, UserRole.ADMIN.value]),
                User.is_active.is_(True),
            )
        )
        return list(result.scalars().all())
```

> Use the `UserRole` enum (already in `src/constants.py`) rather than raw `"owner"`/`"admin"` literals — consistent with the `MetaPlatform`/`MetaConnectionStatus` refactor.

- [ ] **Step 2: Verify import**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/python -c "from src.db.repositories.user import UserRepository; print(hasattr(UserRepository, 'get_admins'))"`
Expected: `True`

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 12: Wire the daily loop into `lifespan`

**Files:**
- Modify: `src/main.py`

- [ ] **Step 1: Import the loop**

Near the top of `src/main.py`, add:

```python
from src.services.meta_token_monitor import meta_token_monitor_loop
```

- [ ] **Step 2: Start + cancel the task in `lifespan`**

In the `lifespan` context manager, after `task = asyncio.create_task(watchdog_loop())`, add:

```python
    meta_task = asyncio.create_task(meta_token_monitor_loop())
```

In the teardown block (after `task.cancel()`), add:

```python
    meta_task.cancel()
```

- [ ] **Step 3: Verify the app still boots**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/python -c "from src.main import app; print('ok')"`
Expected: `ok`

- [ ] **Step 4: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 13: Integration test — refresh attempted + success resets warning

> **As-built test pattern (AUTHORITATIVE for Tasks 13–16 — the code blocks in these tasks are pre-refactor: they use the old flat table, the functional API, and `obfuscate`/`deobfuscate`). Apply these corrections to every test:**
>
> - **Call surface:** `await MetaTokenMonitor(db_session, fake_client).process_agency_tokens(now)` (class-based; inject the fake graph client into the constructor — like the `MetaSocialService` tests). Import `MetaTokenMonitor`, `WARN_5`, `WARN_14` from `src.services.meta_token_monitor`.
> - **Token codec:** `encrypt`/`decrypt` from `src.utils` (the test env / conftest must set `TOKEN_ENCRYPTION_KEY`).
> - **Two-table seeding:** build an `AgencySocialConnection` (carrying the lifecycle fields) and append a child `AgencySocialAccount`. The connection has **no** `platform`/`page_id`/`page_name`/`is_primary` columns — those live on the account.
> - **Selecting the row:** `select(AgencySocialConnection).where(AgencySocialConnection.agency_id == ...)` — **never** `AgencySocialConnection.is_primary` (not a column on the connection).
> - Corrected seed helper:
>   ```python
>   async def _seed_agency(session, agency_id, days_left, now):
>       conn = AgencySocialConnection(
>           agency_id=agency_id,
>           user_access_token=encrypt("old-user-token"),
>           token_expires_at=now + timedelta(days=days_left),
>           status=MetaConnectionStatus.CONNECTED.value,
>           warning_level=0,
>       )
>       conn.accounts.append(AgencySocialAccount(
>           agency_id=agency_id, platform=MetaPlatform.FACEBOOK.value,
>           provider_id="PAGE1", access_token=encrypt("page-token"),
>           is_primary=True, authorized=True,
>       ))
>       session.add(conn)
>       await session.flush()
>   ```

**Files:**
- Create: `tests/integration/test_meta_token_monitor.py`

- [ ] **Step 1: Write the failing test**

Create `tests/integration/test_meta_token_monitor.py`:

```python
from datetime import datetime, timedelta, timezone

import pytest
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.constants import MetaConnectionStatus, MetaPlatform
from src.utils import deobfuscate, obfuscate
from src.models.agency_social_connection import AgencySocialConnection
from src.services import meta_token_monitor
from src.services.meta_token_monitor import (
    WARN_5,
    WARN_14,
    process_agency_tokens,
)


class FakeGraphClient:
    def __init__(self, refreshed=None, raise_error=False):
        self._refreshed = refreshed or {
            "access_token": "fresh-user-token",
            "expires_in": 60 * 24 * 60 * 60,
        }
        self._raise = raise_error
        self.refresh_calls = 0

    async def refresh_long_lived(self, token):
        self.refresh_calls += 1
        if self._raise:
            from src.integrations.meta_graph import MetaGraphError

            raise MetaGraphError("boom")
        return self._refreshed

    async def get_pages(self, user_token):
        return [{"id": "PAGE1", "name": "Page One", "access_token": "page-token"}]


async def _seed_agency(
    session: AsyncSession, agency_id: str, days_left: int, now: datetime
):
    expires = now + timedelta(days=days_left)
    session.add(
        AgencySocialConnection(
            agency_id=agency_id,
            platform=MetaPlatform.FACEBOOK.value,
            page_id="PAGE1",
            page_name="Page One",
            access_token=obfuscate("page-token"),
            is_primary=True,
            user_access_token=obfuscate("old-user-token"),
            token_expires_at=expires,
            warning_level=0,
            status=MetaConnectionStatus.CONNECTED.value,
        )
    )
    await session.flush()


@pytest.mark.asyncio
async def test_refresh_success_resets_state(db_session: AsyncSession):
    now = datetime.now(timezone.utc)
    await _seed_agency(db_session, "default", days_left=10, now=now)
    client = FakeGraphClient()

    await process_agency_tokens(db_session, client, now)

    assert client.refresh_calls == 1
    row = (
        await db_session.execute(
            select(AgencySocialConnection).where(
                AgencySocialConnection.agency_id == "default"
            )
        )
    ).scalar_one()
    assert deobfuscate(row.user_access_token) == "fresh-user-token"
    assert row.warning_level == 0
    assert row.status == MetaConnectionStatus.CONNECTED.value
    assert (row.token_expires_at - now).days > WARN_14
```

- [ ] **Step 2: Run it to verify it fails (table/logic wired)**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/pytest tests/integration/test_meta_token_monitor.py::test_refresh_success_resets_state -v`
Expected: PASS once the service from Task 10 is correct. If it FAILS, read the assertion and fix `meta_token_monitor.py` (this is the implementation-verifying test for the success path).

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 14: Integration test — hard refresh failure ⇒ needs_reconnect (in-app)

> **As-built — there is no notify to assert; Step 2's dedupe-guard is obsolete.** The monitor no longer calls `notify_reconnect` (in-app notification only — Task 10 as-built), so **drop the `monkeypatch`/`calls` machinery in Step 1 and Step 2 entirely**. The test just asserts the in-app status after a hard failure (writing `needs_reconnect` is idempotent across ticks, so no guard is needed):
> ```python
> @pytest.mark.asyncio
> async def test_refresh_failure_sets_needs_reconnect(db_session):
>     now = datetime.now(timezone.utc)
>     await _seed_agency(db_session, "default", days_left=3, now=now)
>     client = FakeGraphClient(raise_error=True)
>     await MetaTokenMonitor(db_session, client).process_agency_tokens(now)
>     row = (await db_session.execute(
>         select(AgencySocialConnection).where(
>             AgencySocialConnection.agency_id == "default"))).scalar_one()
>     assert row.status == MetaConnectionStatus.NEEDS_RECONNECT.value
>     assert row.last_error == "silent refresh failed"
> ```

**Files:**
- Modify: `tests/integration/test_meta_token_monitor.py`

- [ ] **Step 1: Write the test (mocking `notify_reconnect`)**

Append:

```python
@pytest.mark.asyncio
async def test_refresh_failure_sets_needs_reconnect_and_notifies_once(
    db_session: AsyncSession, monkeypatch
):
    now = datetime.now(timezone.utc)
    await _seed_agency(db_session, "default", days_left=3, now=now)
    client = FakeGraphClient(raise_error=True)

    calls = {"n": 0}

    async def fake_notify(session, agency_id, connected_by_user_id):
        calls["n"] += 1

    monkeypatch.setattr(meta_token_monitor, "notify_reconnect", fake_notify)

    await process_agency_tokens(db_session, client, now)
    await process_agency_tokens(db_session, client, now)

    row = (
        await db_session.execute(
            select(AgencySocialConnection).where(
                AgencySocialConnection.agency_id == "default"
            )
        )
    ).scalar_one()
    assert row.status == MetaConnectionStatus.NEEDS_RECONNECT.value
    assert calls["n"] == 1
```

> Note: dedupe-to-once works because after the first tick `status` is `needs_reconnect`; the second tick still attempts refresh (still ≤14 days) and fails again, so to guarantee "once" the implementation must skip notifying when `status` is already `needs_reconnect`. Add that guard in `_process_one_agency`'s hard-failure branch: only call `notify_reconnect` if `primary.status != MetaConnectionStatus.NEEDS_RECONNECT.value`. Update `src/services/meta_token_monitor.py` accordingly, then re-run.

- [ ] **Step 2: Add the dedupe guard to the service**

In `_process_one_agency`, change the `except MetaGraphError:` branch (from Task 10) to guard the notify, keeping the repository + enum form:

```python
        except MetaGraphError:
            if primary.status != MetaConnectionStatus.NEEDS_RECONNECT.value:
                await notify_reconnect(
                    session, agency_id, primary.connected_by_user_id
                )
            await repo.update_by_agency(
                agency_id,
                status=MetaConnectionStatus.NEEDS_RECONNECT.value,
                last_error="silent refresh failed",
            )
            return
```

- [ ] **Step 3: Run the test**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/pytest tests/integration/test_meta_token_monitor.py::test_refresh_failure_sets_needs_reconnect_and_notifies_once -v`
Expected: PASS.

- [ ] **Step 4: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 15: Integration test — warning thresholds (14 → 5 → expired) + no-refresh-needed

**Files:**
- Modify: `tests/integration/test_meta_token_monitor.py`

- [ ] **Step 1: Write the tests**

Append:

```python
class NoExtendGraphClient(FakeGraphClient):
    def __init__(self, days_left, now):
        super().__init__(
            refreshed={
                "access_token": "same-window-token",
                "expires_in": days_left * 24 * 60 * 60,
            }
        )


@pytest.mark.asyncio
async def test_14_day_warning_then_5_day_then_expired(db_session: AsyncSession):
    now = datetime.now(timezone.utc)
    await _seed_agency(db_session, "default", days_left=12, now=now)
    client = NoExtendGraphClient(days_left=12, now=now)

    await process_agency_tokens(db_session, client, now)
    row = (
        await db_session.execute(
            select(AgencySocialConnection).where(
                AgencySocialConnection.is_primary.is_(True)
            )
        )
    ).scalar_one()
    assert row.warning_level == WARN_14
    assert row.status == MetaConnectionStatus.EXPIRING.value

    row.token_expires_at = now + timedelta(days=4)
    await db_session.flush()
    client2 = NoExtendGraphClient(days_left=4, now=now)
    await process_agency_tokens(db_session, client2, now)
    await db_session.refresh(row)
    assert row.warning_level == WARN_5

    row.token_expires_at = now - timedelta(days=1)
    await db_session.flush()
    client3 = NoExtendGraphClient(days_left=-1, now=now)
    await process_agency_tokens(db_session, client3, now)
    await db_session.refresh(row)
    assert row.status == MetaConnectionStatus.EXPIRED.value


@pytest.mark.asyncio
async def test_no_action_when_token_healthy(db_session: AsyncSession):
    now = datetime.now(timezone.utc)
    await _seed_agency(db_session, "default", days_left=40, now=now)
    client = FakeGraphClient()

    await process_agency_tokens(db_session, client, now)

    assert client.refresh_calls == 0
    row = (
        await db_session.execute(
            select(AgencySocialConnection).where(
                AgencySocialConnection.is_primary.is_(True)
            )
        )
    ).scalar_one()
    assert row.warning_level == 0
    assert row.status == MetaConnectionStatus.CONNECTED.value
```

- [ ] **Step 2: Run the tests**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/pytest tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS. Fix `meta_token_monitor.py` threshold logic if any fail.

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 16: Integration test — per-agency isolation

**Files:**
- Modify: `tests/integration/test_meta_token_monitor.py`

- [ ] **Step 1: Write the test**

Append:

```python
@pytest.mark.asyncio
async def test_two_agencies_isolated(db_session: AsyncSession):
    now = datetime.now(timezone.utc)
    await _seed_agency(db_session, "agency-a", days_left=40, now=now)
    await _seed_agency(db_session, "agency-b", days_left=3, now=now)
    client = FakeGraphClient()

    await process_agency_tokens(db_session, client, now)

    rows = (
        await db_session.execute(select(AgencySocialConnection))
    ).scalars().all()
    by_agency = {r.agency_id: r for r in rows}
    assert by_agency["agency-a"].status == MetaConnectionStatus.CONNECTED.value
    assert by_agency["agency-a"].warning_level == 0
    assert by_agency["agency-b"].status in (
        MetaConnectionStatus.EXPIRING.value,
        MetaConnectionStatus.CONNECTED.value,
    )
```

- [ ] **Step 2: Run the full job test file**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/pytest tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS.

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

## Milestone 4 — Reconnect notification verification (in-app)

Reconnect is surfaced **in-app, not by email** (Task 10 as-built). The monitor writes `status=needs_reconnect` on the connection; the Meta settings panel renders `frontend/src/components/integrations/MetaStatusBanner.tsx`. The status-writing logic is covered by the Task 14 integration test — this milestone manually verifies the **UI surface**.

### Task 17: Manual smoke test of the in-app reconnect banner

- [ ] **Step 1: Force the connection into `needs_reconnect`**

The monitor's status-writing is covered by the Task 14 test, so set the status directly to isolate the UI check (the local Docker DB; one connection per agency, keyed by `agency_id`, **not** `is_primary`):

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c \
  "UPDATE agency_social_connections SET status='needs_reconnect', last_error='silent refresh failed' WHERE agency_id='default';"
```

- [ ] **Step 2: Confirm the in-app banner**

Open Integrations → Meta (`/integrations?tab=meta`). Expected: a **red** banner above the cards — "Your Meta connection needs to be reconnected…". Set `status='expiring'` instead and reload → an **amber** "expiring soon" banner. (`MetaConnectionResponse.status` → `MetaStatusBanner`.)

- [ ] **Step 3: Clear it**

Click **Reconnect** on the card (or `UPDATE … SET status='connected'`) → reload → the banner disappears. _(SKIP commit per execution override — leave any fixes in the working tree.)_

---

### Task 18: Extract shared test doubles + factories into dedicated `tests/fakes/` + `tests/factories/` folders

> **Why:** `FakeGraphClient`/`NoExtendGraphClient` (test doubles) and `seed_agency` (data factory) were written inline in `tests/integration/test_meta_token_monitor.py`. They'll be reused by upcoming Meta/social tests (service, API). Keep `conftest.py` for fixtures/wiring only; put importable helper classes/builders in dedicated, purpose-named packages — `tests/fakes/` for doubles, `tests/factories/` for data builders, domain-named files inside each — so they're greppable and shared across the unit/integration tiers.

**Files:**
- Create: `tests/fakes/__init__.py` (empty)
- Create: `tests/fakes/meta_graph.py` — `FakeGraphClient`, `NoExtendGraphClient`, `FRESH_USER_TOKEN`, `SECONDS_PER_DAY`
- Create: `tests/factories/__init__.py` (empty)
- Create: `tests/factories/social.py` — `async def seed_agency(session, agency_id, days_left, now) -> AgencySocialConnection`
- Modify: `tests/integration/test_meta_token_monitor.py` — delete the inline doubles + seed helper; import them from `tests.fakes.meta_graph` / `tests.factories.social`

- [ ] **Step 1: Create `tests/fakes/meta_graph.py`** — move `FakeGraphClient`/`NoExtendGraphClient` verbatim (they only implement `refresh_long_lived`, the one method the monitor calls). Keep `SECONDS_PER_DAY`/`FRESH_USER_TOKEN` here so assertions can import the canonical value.

- [ ] **Step 2: Create `tests/factories/social.py`** — move `seed_agency`; have it `return connection` for convenience. Two-table seed (connection + child `AgencySocialAccount`), `encrypt(...)` tokens — never `is_primary` on the connection.

- [ ] **Step 3: Slim the test module** — replace the inline definitions with:
  ```python
  from tests.fakes.meta_graph import FakeGraphClient, NoExtendGraphClient, FRESH_USER_TOKEN
  from tests.factories.social import seed_agency
  ```
  Update the call sites (`await seed_agency(db_session, ...)` — unchanged signature). Keep the local `_load` helper (it's specific to these assertions).

- [ ] **Step 4: Run the tests (Docker — no host venv; the slim image has no pytest, install it ad-hoc)**

  ```bash
  docker compose -f docker-compose.local.yml exec -T backend \
    python -m pytest tests/integration/test_meta_token_monitor.py -v -p no:cacheprovider
  ```

  Expected: 5 passed. (The `__init__.py` in each new folder makes them importable packages alongside the existing `tests/__init__.py`.)

- [ ] **Step 5: Commit** _(SKIP per execution override — leave in working tree)_

---

## Milestone 5 — Connected-card detail + real-time amber/red expiry states (UI polish to fully meet the spec)

> **Why this milestone exists (gap against the spec).** The shipped Meta tab connects, gates Instagram behind Facebook, opens the OAuth popup, and shows a Connected / Not-connected pill — but it is **missing three spec requirements**:
>
> 1. **"On successful OAuth: update the card to show the connected Page/account name, connection date, and token expiry."** Today the card shows only a pill + button. The data is already in the API response — `MetaConnectionResponse` exposes `connected_at`, `token_expires_at`, and per-account `name`/`username` — it's just never rendered.
> 2. **"Show clear warning state when token is expiring within 14 days (amber) or 5 days (red)."** The current `MetaStatusBanner` shows a single amber message for `status="expiring"` and does **not** distinguish 14-day (amber) from 5-day (red). It's also driven only by the daily-monitor `status`, which lags reality between ticks.
> 3. **"Expiring imminently (< 5 days) — red warning with prominent reconnect CTA"** and **"Disconnected / expired — error state with reconnect prompt."** There is no prominent/red reconnect CTA and the cards don't reflect the expired/needs-reconnect state at all.
>
> **Approach.** Derive a single `MetaHealth` state on the frontend by combining **`token_expires_at` (real-time day math)** with the backend **`status`** (which alone can express `needs_reconnect`/`error` — states not derivable from a date). The card and the banner both consume that one derived value, so the amber/red thresholds are correct the instant the page loads, independent of when the daily monitor last ran. Thresholds match the backend monitor exactly: **≤14 days → amber, ≤5 days → red, ≤0 → expired**.
>
> **No backend or schema change.** Every field is already returned by `GET /api/social/meta/connection`. This milestone is **frontend-only** and, like Milestones 1/2/4, is **verified manually** (the repo has no frontend test runner or date library — confirmed). All code below MUST satisfy `frontend/CLAUDE.md` (no comments, no `as`/`!`, constants `UPPER_SNAKE_CASE`, no inline arrow handlers in JSX props, no nested ternaries, shared types in `types/meta.ts`).

### Task 19: Health-state types + `deriveMetaHealth` helper (pure logic)

**Files:**
- Modify: `frontend/src/types/meta.ts`
- Create: `frontend/src/lib/metaHealth.ts`

- [ ] **Step 1: Add the health types**

Append to `frontend/src/types/meta.ts` (the existing file already exports `MetaPlatform`, `MetaAccount`, `MetaConnection`, `MetaConnectResponse` — keep them):

```ts
export type MetaHealthLevel =
  | "none"
  | "healthy"
  | "expiring"
  | "imminent"
  | "expired"
  | "needs_reconnect";

export interface MetaHealth {
  level: MetaHealthLevel;
  daysRemaining: number | null;
}
```

- [ ] **Step 2: Write the helper**

Create `frontend/src/lib/metaHealth.ts`:

```ts
import type {
  MetaConnection,
  MetaHealth,
  MetaHealthLevel,
} from "../types/meta";

const EXPIRY_AMBER_DAYS = 14;
const EXPIRY_RED_DAYS = 5;
const MS_PER_DAY = 1000 * 60 * 60 * 24;

const STATUS_NEEDS_RECONNECT = "needs_reconnect";
const STATUS_EXPIRED = "expired";
const STATUS_ERROR = "error";

export function daysUntil(iso: string): number {
  const target = new Date(iso).getTime();
  return Math.floor((target - Date.now()) / MS_PER_DAY);
}

export function formatConnectionDate(iso: string): string {
  return new Date(iso).toLocaleDateString();
}

function levelFromDays(days: number): MetaHealthLevel {
  if (days <= EXPIRY_RED_DAYS) {
    return "imminent";
  }
  if (days <= EXPIRY_AMBER_DAYS) {
    return "expiring";
  }
  return "healthy";
}

export function deriveMetaHealth(
  connection: MetaConnection | null | undefined,
): MetaHealth {
  if (!connection || connection.accounts.length === 0) {
    return { level: "none", daysRemaining: null };
  }
  if (
    connection.status === STATUS_NEEDS_RECONNECT ||
    connection.status === STATUS_ERROR
  ) {
    return { level: "needs_reconnect", daysRemaining: null };
  }
  if (!connection.token_expires_at) {
    if (connection.status === STATUS_EXPIRED) {
      return { level: "expired", daysRemaining: null };
    }
    return { level: "healthy", daysRemaining: null };
  }
  const days = daysUntil(connection.token_expires_at);
  if (connection.status === STATUS_EXPIRED || days <= 0) {
    return { level: "expired", daysRemaining: days };
  }
  return { level: levelFromDays(days), daysRemaining: days };
}
```

- [ ] **Step 3: Verify it type-checks**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit`
Expected: no errors referencing `metaHealth.ts` or `MetaHealth`.

- [ ] **Step 4: Verify the threshold boundaries by hand**

There is no frontend test runner, so eyeball `levelFromDays` + `deriveMetaHealth` against this table (these are the exact boundaries the spec calls out and they match the backend monitor's `WARN_14`/`WARN_5`/`≤0`):

| `token_expires_at` vs now   | `status`          | expected `level`  | expected pill colour |
| --------------------------- | ----------------- | ----------------- | -------------------- |
| no connection / no accounts | —                 | `none`            | grey                 |
| +40 days                    | `connected`       | `healthy`         | green                |
| +15 days                    | `connected`       | `healthy`         | green                |
| +14 days                    | `connected`       | `expiring`        | amber                |
| +6 days                     | `connected`       | `expiring`        | amber                |
| +5 days                     | `connected`       | `imminent`        | red                  |
| +1 day                      | `connected`       | `imminent`        | red                  |
| 0 / past                    | `connected`       | `expired`         | red                  |
| any                         | `expired`         | `expired`         | red                  |
| any                         | `needs_reconnect` | `needs_reconnect` | red                  |
| any                         | `error`           | `needs_reconnect` | red                  |

- [ ] **Step 5: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 20: `MetaAccountCard` — render connected detail + health pill + prominent reconnect CTA

**Files:**
- Modify: `frontend/src/components/integrations/MetaAccountCard.tsx`

- [ ] **Step 1: Replace the card with the detail-aware version**

Overwrite `frontend/src/components/integrations/MetaAccountCard.tsx` with the full file below. New optional props (`accountName`, `connectedAt`, `expiresAt`, `health`) are additive — the panel (Task 22) supplies them. When `connected` is false the card ignores `health` and shows the grey "Not connected" pill + primary "Connect" button (so the Instagram card stays in its not-connected state even while the Facebook connection is expiring). `MetaConnectedDetail` is a file-local presentational helper (not exported) to keep the card's JSX flat:

```tsx
import type { MetaHealth, MetaHealthLevel } from "../../types/meta";
import { formatConnectionDate } from "../../lib/metaHealth";

type HealthColor = "green" | "amber" | "red" | "gray";

interface MetaAccountCardProps {
  title: string;
  subtitle: string;
  iconBg: string;
  iconChar: string;
  connected: boolean;
  disabled: boolean;
  buttonLabel: string;
  onConnect: () => void;
  accountName?: string | null;
  connectedAt?: string | null;
  expiresAt?: string | null;
  health?: MetaHealth;
}

interface MetaConnectedDetailProps {
  accountName?: string | null;
  connectedAt?: string | null;
  expiresAt?: string | null;
  expiryClass: string;
  expirySuffixText: string;
}

const RECONNECT_LABEL = "Reconnect";
const NOT_CONNECTED_LABEL = "Not connected";
const CONNECTED_LABEL = "Connected";
const EXPIRED_LABEL = "Expired";
const RECONNECT_NEEDED_LABEL = "Reconnect needed";

const PILL_BASE =
  "inline-flex items-center gap-1.5 rounded-full px-2.5 py-1 text-xs font-medium";
const PILL_BY_COLOR: Record<HealthColor, string> = {
  green: "bg-green-50 text-green-700",
  amber: "bg-amber-50 text-amber-700",
  red: "bg-red-50 text-red-700",
  gray: "bg-gray-100 text-gray-600",
};
const DOT_BY_COLOR: Record<HealthColor, string> = {
  green: "bg-green-500",
  amber: "bg-amber-500",
  red: "bg-red-500",
  gray: "bg-gray-400",
};
const EXPIRY_TEXT_BY_COLOR: Record<HealthColor, string> = {
  green: "text-gray-600",
  amber: "text-amber-700 font-medium",
  red: "text-red-700 font-medium",
  gray: "text-gray-600",
};

const CARD_BASE = "rounded-xl border p-6 shadow-sm";
const CARD_ACTIVE = "border-gray-200 bg-white";
const CARD_MUTED = "border-gray-200 bg-gray-50 opacity-75";

const BUTTON_BASE =
  "mt-4 rounded-lg px-4 py-2 text-sm font-medium disabled:cursor-not-allowed disabled:opacity-50";
const BUTTON_PRIMARY = "bg-indigo-600 text-white hover:bg-indigo-700";
const BUTTON_SECONDARY =
  "border border-gray-300 bg-white text-gray-700 hover:bg-gray-50";
const BUTTON_DANGER = "bg-red-600 text-white hover:bg-red-700";

const PROMINENT_LEVELS: MetaHealthLevel[] = [
  "imminent",
  "expired",
  "needs_reconnect",
];

function effectiveLevel(
  connected: boolean,
  health?: MetaHealth,
): MetaHealthLevel {
  if (!connected) {
    return "none";
  }
  return health?.level ?? "healthy";
}

function colorForLevel(level: MetaHealthLevel): HealthColor {
  if (level === "healthy") {
    return "green";
  }
  if (level === "expiring") {
    return "amber";
  }
  if (level === "none") {
    return "gray";
  }
  return "red";
}

function pillLabelForLevel(level: MetaHealthLevel, days: number | null): string {
  if (level === "none") {
    return NOT_CONNECTED_LABEL;
  }
  if (level === "healthy") {
    return CONNECTED_LABEL;
  }
  if (level === "expired") {
    return EXPIRED_LABEL;
  }
  if (level === "needs_reconnect") {
    return RECONNECT_NEEDED_LABEL;
  }
  return `Expires in ${days}d`;
}

function buttonClassForState(connected: boolean, prominent: boolean): string {
  if (!connected) {
    return BUTTON_PRIMARY;
  }
  if (prominent) {
    return BUTTON_DANGER;
  }
  return BUTTON_SECONDARY;
}

function expirySuffix(days: number | null): string {
  if (days === null || days <= 0) {
    return "";
  }
  return ` (in ${days} days)`;
}

function MetaConnectedDetail(props: MetaConnectedDetailProps) {
  return (
    <div className="mt-4 space-y-1 text-sm">
      {props.accountName && (
        <p className="font-medium text-gray-800">{props.accountName}</p>
      )}
      {props.connectedAt && (
        <p className="text-gray-600">
          Connected {formatConnectionDate(props.connectedAt)}
        </p>
      )}
      {props.expiresAt && (
        <p className={props.expiryClass}>
          Token expires {formatConnectionDate(props.expiresAt)}
          {props.expirySuffixText}
        </p>
      )}
    </div>
  );
}

export function MetaAccountCard(props: MetaAccountCardProps) {
  const level = effectiveLevel(props.connected, props.health);
  const days = props.health?.daysRemaining ?? null;
  const color = colorForLevel(level);
  const prominent = PROMINENT_LEVELS.includes(level);
  const buttonLabel = props.connected ? RECONNECT_LABEL : props.buttonLabel;
  const buttonClass = buttonClassForState(props.connected, prominent);
  const muted = props.disabled && !props.connected;
  const cardClass = muted ? CARD_MUTED : CARD_ACTIVE;
  return (
    <div className={`${CARD_BASE} ${cardClass}`}>
      <div className="flex items-center justify-between">
        <div className="flex items-center gap-4">
          <div
            className={`flex h-10 w-10 items-center justify-center rounded-lg text-lg font-bold text-white ${props.iconBg}`}
          >
            {props.iconChar}
          </div>
          <div>
            <p className="font-semibold text-gray-900">{props.title}</p>
            <p className="text-sm text-gray-500">{props.subtitle}</p>
          </div>
        </div>
        <span className={`${PILL_BASE} ${PILL_BY_COLOR[color]}`}>
          <span className={`h-1.5 w-1.5 rounded-full ${DOT_BY_COLOR[color]}`} />
          {pillLabelForLevel(level, days)}
        </span>
      </div>
      {props.connected && (
        <MetaConnectedDetail
          accountName={props.accountName}
          connectedAt={props.connectedAt}
          expiresAt={props.expiresAt}
          expiryClass={EXPIRY_TEXT_BY_COLOR[color]}
          expirySuffixText={expirySuffix(days)}
        />
      )}
      <button
        type="button"
        onClick={props.onConnect}
        disabled={props.disabled}
        className={`${BUTTON_BASE} ${buttonClass}`}
      >
        {buttonLabel}
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Verify it type-checks and builds**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit`
Expected: no errors. (The panel still passes the old prop set until Task 22 — `accountName`/`connectedAt`/`expiresAt`/`health` are optional, so the existing call sites keep compiling.)

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 21: `MetaStatusBanner` — amber (≤14d) / red (≤5d) via `MetaHealth` + reconnect CTA

**Files:**
- Modify: `frontend/src/components/integrations/MetaStatusBanner.tsx`

- [ ] **Step 1: Replace the banner with the health-driven version**

Overwrite `frontend/src/components/integrations/MetaStatusBanner.tsx`. It now takes the derived `MetaHealth` plus an `onReconnect` callback. Amber for `expiring`; red + a prominent **Reconnect** button for `imminent`/`expired`/`needs_reconnect`; nothing for `none`/`healthy`:

```tsx
import type { MetaHealth, MetaHealthLevel } from "../../types/meta";

const BANNER_BASE =
  "flex items-center justify-between gap-4 rounded-lg px-4 py-3 text-sm";
const BANNER_RED = "bg-red-50 text-red-700";
const BANNER_AMBER = "bg-amber-50 text-amber-700";
const RECONNECT_BUTTON =
  "shrink-0 rounded-lg bg-red-600 px-3 py-1.5 text-xs font-medium text-white hover:bg-red-700";

const RED_LEVELS: MetaHealthLevel[] = [
  "imminent",
  "expired",
  "needs_reconnect",
];

const MESSAGE_NEEDS_RECONNECT =
  "Your Meta connection needs to be reconnected. Reconnect to restore posting to Facebook and Instagram.";
const MESSAGE_EXPIRED =
  "Your Meta connection has expired. Reconnect to restore posting to Facebook and Instagram.";

interface MetaStatusBannerProps {
  health: MetaHealth;
  onReconnect: () => void;
}

function bannerMessage(health: MetaHealth): string {
  if (health.level === "needs_reconnect") {
    return MESSAGE_NEEDS_RECONNECT;
  }
  if (health.level === "expired") {
    return MESSAGE_EXPIRED;
  }
  if (health.level === "imminent") {
    return `Your Meta connection expires in ${health.daysRemaining} days. Reconnect now to avoid an interruption.`;
  }
  return `Your Meta connection expires in ${health.daysRemaining} days. Reconnect soon to avoid an interruption.`;
}

export function MetaStatusBanner({ health, onReconnect }: MetaStatusBannerProps) {
  if (health.level === "none" || health.level === "healthy") {
    return null;
  }
  const isRed = RED_LEVELS.includes(health.level);
  const tone = isRed ? BANNER_RED : BANNER_AMBER;
  return (
    <div className={`${BANNER_BASE} ${tone}`}>
      <span>{bannerMessage(health)}</span>
      {isRed && (
        <button type="button" onClick={onReconnect} className={RECONNECT_BUTTON}>
          Reconnect
        </button>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Verify type-check**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit`
Expected: this will report an error in `MetaSettingsPage.tsx` (still passes `status={...}` to the banner) — that's expected and fixed in Task 22. No error inside `MetaStatusBanner.tsx` itself.

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 22: `MetaSettingsPanel` — wire health + per-card detail; align card copy to the spec

**Files:**
- Modify: `frontend/src/pages/MetaSettingsPage.tsx`

- [ ] **Step 1: Replace the panel with the wired-up version**

Overwrite `frontend/src/pages/MetaSettingsPage.tsx`. It derives `health` once via `deriveMetaHealth`, resolves the Facebook and Instagram accounts from the connection's nested `accounts`, and feeds each card its name + the connection-level date/expiry + the shared `health`. Card titles now read "Facebook Page" / "Instagram Business" to match the spec, and the Instagram subtitle reflects its connected state. Reconnect (card or banner) re-runs the Facebook connect, which refreshes the user token:

```tsx
import { useEffect } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import {
  fetchMetaConnection,
  metaConnectFacebook,
  metaConnectInstagram,
} from "../api/metaCredentials";
import type { MetaConnectResponse } from "../types/meta";
import { deriveMetaHealth } from "../lib/metaHealth";
import { MetaAccountCard } from "../components/integrations/MetaAccountCard";
import { MetaStatusBanner } from "../components/integrations/MetaStatusBanner";

const META_OAUTH_COMPLETE = "meta-oauth-complete";
const POPUP_FEATURES = "width=600,height=700";
const POPUP_NAME = "meta-oauth";
const CONNECTION_KEY = ["meta-connection"];
const PLATFORM_FACEBOOK = "facebook";
const PLATFORM_INSTAGRAM = "instagram";

const FACEBOOK_TITLE = "Facebook Page";
const FACEBOOK_SUBTITLE = "Publish posts to your Facebook Page";
const FACEBOOK_BUTTON = "Connect Facebook";
const INSTAGRAM_TITLE = "Instagram Business";
const INSTAGRAM_BUTTON = "Connect Instagram";
const INSTAGRAM_SUBTITLE_CONNECTED =
  "Publish posts to your Instagram Business account";
const INSTAGRAM_SUBTITLE_DISCONNECTED = "Connect your Facebook Page first";

function openBlankPopup() {
  return window.open("", POPUP_NAME, POPUP_FEATURES);
}

function navigatePopup(popup: Window | null, data: MetaConnectResponse) {
  if (!popup) {
    return;
  }
  popup.location.href = data.authorize_url;
}

export function MetaSettingsPanel() {
  const queryClient = useQueryClient();
  const { data: connection } = useQuery({
    queryKey: CONNECTION_KEY,
    queryFn: fetchMetaConnection,
  });

  const facebookMutation = useMutation({ mutationFn: metaConnectFacebook });
  const instagramMutation = useMutation({ mutationFn: metaConnectInstagram });

  useEffect(() => {
    function handler(event: MessageEvent) {
      if (event.data === META_OAUTH_COMPLETE) {
        queryClient.invalidateQueries({ queryKey: CONNECTION_KEY });
      }
    }
    window.addEventListener("message", handler);
    return () => window.removeEventListener("message", handler);
  }, [queryClient]);

  const accounts = connection?.accounts ?? [];
  const facebookAccount = accounts.find((a) => a.platform === PLATFORM_FACEBOOK);
  const instagramAccount = accounts.find(
    (a) => a.platform === PLATFORM_INSTAGRAM,
  );
  const facebookConnected = Boolean(facebookAccount);
  const instagramConnected = Boolean(instagramAccount);
  const health = deriveMetaHealth(connection);
  const instagramName = instagramAccount?.username ?? instagramAccount?.name;
  const instagramSubtitle = instagramConnected
    ? INSTAGRAM_SUBTITLE_CONNECTED
    : INSTAGRAM_SUBTITLE_DISCONNECTED;

  function handleConnectFacebook() {
    const popup = openBlankPopup();
    facebookMutation.mutate(undefined, {
      onSuccess: (data) => navigatePopup(popup, data),
      onError: () => popup?.close(),
    });
  }
  function handleConnectInstagram() {
    const popup = openBlankPopup();
    instagramMutation.mutate(undefined, {
      onSuccess: (data) => navigatePopup(popup, data),
      onError: () => popup?.close(),
    });
  }

  return (
    <div className="px-8 py-6">
      <div className="mx-auto max-w-2xl space-y-5">
        <div>
          <h3 className="font-semibold text-gray-900">Connected accounts</h3>
          <p className="text-sm text-gray-500">
            Connect your Facebook Page first — Instagram requires a linked
            Facebook Page.
          </p>
        </div>
        <MetaStatusBanner health={health} onReconnect={handleConnectFacebook} />
        <MetaAccountCard
          title={FACEBOOK_TITLE}
          subtitle={FACEBOOK_SUBTITLE}
          iconBg="bg-blue-600"
          iconChar="f"
          connected={facebookConnected}
          disabled={false}
          buttonLabel={FACEBOOK_BUTTON}
          onConnect={handleConnectFacebook}
          accountName={facebookAccount?.name}
          connectedAt={connection?.connected_at}
          expiresAt={connection?.token_expires_at}
          health={health}
        />
        <MetaAccountCard
          title={INSTAGRAM_TITLE}
          subtitle={instagramSubtitle}
          iconBg="bg-pink-500"
          iconChar="IG"
          connected={instagramConnected}
          disabled={!facebookConnected}
          buttonLabel={INSTAGRAM_BUTTON}
          onConnect={handleConnectInstagram}
          accountName={instagramName}
          connectedAt={connection?.connected_at}
          expiresAt={connection?.token_expires_at}
          health={health}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify type-check and build**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit && npm run build`
Expected: no type errors; build succeeds.

- [ ] **Step 3: Commit** _(SKIP per execution override — leave in working tree)_

---

### Task 23: Manual verification of all five states (Milestone 5)

> The five spec states are driven by `token_expires_at` (amber/red/expired computed real-time on the frontend) and `status` (`needs_reconnect`). Because the amber/red thresholds are derived from the expiry date, you can exercise them by editing `token_expires_at` directly — **no need to wait for the daily monitor**. All updates target the single per-agency `agency_social_connections` row (keyed by `agency_id='default'`, the local dev agency), mirroring Task 17. Run the frontend (`cd frontend && npm run dev`) and the Docker backend, open Integrations → Meta (`/integrations?tab=meta`), and reload after each `UPDATE`.

- [ ] **Step 1: Not connected**

With no connection row (`DELETE FROM agency_social_connections WHERE agency_id='default';`), reload. Expected: both cards show a grey **Not connected** pill, no detail rows, no banner; the Facebook button is the primary "Connect Facebook"; the Instagram card is **disabled and visibly greyed/muted** (`CARD_MUTED` — faded background + reduced opacity) with a disabled button.

- [ ] **Step 2: Connected (healthy) — proves the name/date/expiry requirement**

Connect Facebook for real (or set a healthy expiry on an existing row):

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c \
  "UPDATE agency_social_connections SET token_expires_at = now() + interval '40 days', status='connected', warning_level=0 WHERE agency_id='default';"
```

Expected on the Facebook card: green **Connected** pill, the **Page name** line, **"Connected {date}"**, and **"Token expires {date} (in 40 days)"** in grey. No banner.

- [ ] **Step 3: Expiring soon (< 14 days) — amber**

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c \
  "UPDATE agency_social_connections SET token_expires_at = now() + interval '10 days', status='connected' WHERE agency_id='default';"
```

Expected: amber pill reading **"Expires in 10d"**, the expiry line in amber, and an **amber banner** ("expires in 10 days … Reconnect soon") with **no** Reconnect button.

- [ ] **Step 4: Expiring imminently (< 5 days) — red + prominent CTA**

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c \
  "UPDATE agency_social_connections SET token_expires_at = now() + interval '3 days', status='connected' WHERE agency_id='default';"
```

Expected: red pill **"Expires in 3d"**, red expiry line, the card's reconnect button rendered as the **red (danger) "Reconnect"**, and a **red banner** with a prominent **Reconnect** button.

- [ ] **Step 5: Expired**

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c \
  "UPDATE agency_social_connections SET token_expires_at = now() - interval '1 day', status='expired' WHERE agency_id='default';"
```

Expected: red **Expired** pill, red banner ("has expired … Reconnect") with the prominent Reconnect button.

- [ ] **Step 6: Disconnected / needs reconnect (refresh-failure state)**

```bash
docker compose -f docker-compose.local.yml exec -T postgres \
  psql -U slicify -d sideline_realestate -c \
  "UPDATE agency_social_connections SET status='needs_reconnect', last_error='silent refresh failed' WHERE agency_id='default';"
```

Expected: red **Reconnect needed** pill, red banner ("needs to be reconnected … Reconnect") with the prominent Reconnect button. Clicking **Reconnect** (card or banner) opens the OAuth popup; completing it returns the row to healthy and clears the banner.

- [ ] **Step 7: Instagram gating still holds**

Confirm the Instagram card stays disabled (greyed/muted via `CARD_MUTED`) with the **"Connect your Facebook Page first"** subtitle while `facebook` is the only connected account, and that an amber/red Facebook expiry does **not** flip the disabled Instagram card's pill (it remains grey "Not connected" until an `instagram` account exists). _(SKIP commit per execution override — leave any fixes in the working tree.)_

---

## Milestone 6 — Tighten silent-refresh trigger to the final day (post-launch change)

> **Why.** Milestone 3 shipped with the silent refresh firing at the **same ≤14-day threshold** as the first expiry warning (`days_remaining <= WARN_14`). Per the as-built honesty caveat in Milestone 3, `fb_exchange_token` does **not** reset the 60-day clock for server-side apps and Page tokens stay valid while the user token does — so refreshing 14 days out mostly re-stored a token with ~the same remaining lifetime and pre-empted the warning path. This change **decouples refresh from the warnings**: keep the 14-day / 5-day **warnings** as the user-facing signal, and only attempt the silent refresh in the **final day** before expiry, when an extension is actually meaningful (and the token is comfortably past Meta's 24h-minimum-age rule). The warning → reconnect backstop is unchanged.

### Task 24: Refresh trigger threshold 14d → 1d (with test)

> **As-built — implemented and verified (tests green; not yet committed — left in working tree per the execution override).** Decouples the refresh trigger from `WARN_14`; the warnings (`WARN_14`/`WARN_5`) and `_apply_status` are untouched. `.days` truncation means `<= REFRESH_THRESHOLD_DAYS` (1) fires once under ~2 days remain (i.e. before the token drops under a day) and on already-expired tokens.

**Files:**
- Modify: `src/services/meta_token_monitor.py`
- Modify: `tests/integration/test_meta_token_monitor.py`

- [x] **Step 1 (RED): pin the new behaviour with a failing test**

Added `test_no_refresh_outside_final_day` to `tests/integration/test_meta_token_monitor.py`: seed a connection with `days_left=10`, run `process_agency_tokens(now)`, assert `client.refresh_calls == 0` and that the row is warned instead (`warning_level == WARN_14`, `status == EXPIRING`). Against the old `<= WARN_14` code this fails with `refresh_calls == 1` (refresh fired at 10 days) — confirmed RED for the right reason.

- [x] **Step 2 (GREEN): introduce a dedicated threshold constant**

In `src/services/meta_token_monitor.py`, add `REFRESH_THRESHOLD_DAYS = 1` next to the `WARN_*` constants and change the trigger in `_process_one_agency`:

```python
days_remaining = (connection.token_expires_at - now).days
if days_remaining <= REFRESH_THRESHOLD_DAYS:   # was: <= WARN_14
    new_expiry = await self._refresh_user_token(connection, now)
```

`WARN_14` is still used by `_apply_status` for the 14-day warning level — only the refresh trigger changed.

- [x] **Step 3: re-tune the two refresh-path tests to the tighter window**

`test_refresh_success_resets_state` and `test_refresh_failure_sets_needs_reconnect` seeded `days_left=10`/`days_left=3`, which no longer trigger a refresh under the 1-day threshold. Changed both to `days_left=1` so they still exercise the refresh path. `test_14_day_warning_then_5_day_then_expired`, `test_no_action_when_token_healthy`, and `test_two_agencies_isolated` pass unchanged (their warning/expiry assertions don't depend on the refresh trigger).

- [x] **Step 4: run the suite (GREEN)**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && DB_USER=slicify DB_PASSWORD=slicify DB_HOST=localhost DB_PORT=5432 DB_DATABASE=sideline_realestate TOKEN_ENCRYPTION_KEY=<fernet-key> .venv/bin/python -m pytest tests/integration/test_meta_token_monitor.py -v`
Expected: `6 passed` (the 5 originals + `test_no_refresh_outside_final_day`).

> **Local-env notes (one-time, discovered during execution):** the committed `.venv/` was stale (built at the old `/Volumes/T7/...` path before the repo moved) — recreate with `/opt/homebrew/bin/python3.12 -m venv .venv` (the pinned deps lack Python 3.14 wheels). `TOKEN_ENCRYPTION_KEY` isn't in `.env`, so generate an ephemeral Fernet key just for the test run (`.venv/bin/python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`). DB creds come from `.env.local` (`slicify`/`slicify@localhost`), passed as env-var overrides.

- [x] **Step 5: Commit** _(SKIP per execution override — leave in working tree)_

> **Manual test SQL (new threshold).** To make the daily job refresh on its next pass, set the expiry under a day; ≥2 days only warns. Local `sideline_realestate` DB only:
>
> ```sql
> -- triggers refresh on next pass (.days = 0 → <= REFRESH_THRESHOLD_DAYS)
> UPDATE agency_social_connections
> SET token_expires_at = now() + interval '12 hours',
>     status = 'connected', warning_level = 0, last_refreshed_at = NULL
> WHERE user_access_token IS NOT NULL;
>
> -- negative check: no refresh, warns only
> -- SET token_expires_at = now() + interval '2 days'
> -- expired: refresh attempted (→ needs_reconnect if the token is invalid)
> -- SET token_expires_at = now() - interval '1 hour'
> ```
>
> The refresh calls Meta's Graph API for real, so `user_access_token` must be a genuine long-lived token for a successful extend; otherwise the row flips to `needs_reconnect`.

---

## Final verification

- [ ] **Run the full job test suite**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/pytest tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS.

- [ ] **Backend boots; frontend builds**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && .venv/bin/python -c "from src.main import app; print('ok')" && cd frontend && npx tsc --noEmit && npm run build`
Expected: `ok` + clean type-check + successful build.

- [ ] **Deploy-time only:** run `scripts/migrate_all_tenants.py` to apply the migration to every tenant + `slate_template` (per `CLAUDE.md`). Do NOT run during local dev.
