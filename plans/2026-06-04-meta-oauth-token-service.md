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

**Frontend (modify):**
- `frontend/src/types/index.ts` — `MetaConnection`, `MetaConnectResponse`.
- `frontend/src/pages/IntegrationsPage.tsx` — add `meta` to `FIXED_TABS` + dispatch the panel.

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

### Task 2: Create the `AgencySocialConnection` model

**Files:**
- Create: `src/models/agency_social_connection.py`
- Modify: `src/models/__init__.py`

- [ ] **Step 1: Write the model**

Create `src/models/agency_social_connection.py`:

```python
import uuid
from datetime import datetime
from typing import Optional

from sqlalchemy import Boolean, DateTime, ForeignKey, SmallInteger, String, Text
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from src.models.base import Base, TimestampMixin, UUIDMixin


class AgencySocialConnection(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "agency_social_connections"

    agency_id: Mapped[Optional[str]] = mapped_column(String(100), index=True)
    platform: Mapped[str] = mapped_column(String(20), nullable=False)
    page_id: Mapped[Optional[str]] = mapped_column(String(100))
    ig_account_id: Mapped[Optional[str]] = mapped_column(String(100))
    page_name: Mapped[Optional[str]] = mapped_column(String(255))
    ig_username: Mapped[Optional[str]] = mapped_column(String(255))
    access_token: Mapped[Optional[str]] = mapped_column(Text)
    is_primary: Mapped[bool] = mapped_column(
        Boolean, nullable=False, server_default="false"
    )
    user_access_token: Mapped[Optional[str]] = mapped_column(Text)
    token_expires_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True)
    )
    warning_level: Mapped[int] = mapped_column(
        SmallInteger, nullable=False, server_default="0"
    )
    status: Mapped[str] = mapped_column(
        String(20), nullable=False, server_default="connected"
    )
    last_error: Mapped[Optional[str]] = mapped_column(Text)
    connected_by_user_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    last_warned_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    connected_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    last_refreshed_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True)
    )
```

- [ ] **Step 2: Register the model so `Base.metadata` includes the table**

In `src/models/__init__.py`, add an import alongside the other model imports (so tests' `Base.metadata.create_all` builds the table):

```python
from src.models.agency_social_connection import AgencySocialConnection  # noqa: F401
```

- [ ] **Step 3: Verify the model imports and the table is registered**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.models import Base; print('agency_social_connections' in Base.metadata.tables)"`
Expected: `True`

- [ ] **Step 4: Commit**

```bash
git add src/models/agency_social_connection.py src/models/__init__.py
git commit -m "feat(meta): add AgencySocialConnection model"
```

---

### Task 3: Alembic migration for `agency_social_connections`

**Files:**
- Create: `alembic/versions/m1a2b3c4d5e6_add_agency_social_connections.py`

- [ ] **Step 1: Write the migration**

Create `alembic/versions/m1a2b3c4d5e6_add_agency_social_connections.py`:

```python
"""Add agency_social_connections (Meta OAuth tokens)

Revision ID: m1a2b3c4d5e6
Revises: z0t1u2v3w4x5
Create Date: 2026-06-04
"""

from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql


revision: str = "m1a2b3c4d5e6"
down_revision: Union[str, None] = "z0t1u2v3w4x5"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.create_table(
        "agency_social_connections",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("agency_id", sa.String(length=100), nullable=True),
        sa.Column("platform", sa.String(length=20), nullable=False),
        sa.Column("page_id", sa.String(length=100), nullable=True),
        sa.Column("ig_account_id", sa.String(length=100), nullable=True),
        sa.Column("page_name", sa.String(length=255), nullable=True),
        sa.Column("ig_username", sa.String(length=255), nullable=True),
        sa.Column("access_token", sa.Text(), nullable=True),
        sa.Column(
            "is_primary", sa.Boolean(), nullable=False, server_default="false"
        ),
        sa.Column("user_access_token", sa.Text(), nullable=True),
        sa.Column("token_expires_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column(
            "warning_level", sa.SmallInteger(), nullable=False, server_default="0"
        ),
        sa.Column(
            "status", sa.String(length=20), nullable=False, server_default="connected"
        ),
        sa.Column("last_error", sa.Text(), nullable=True),
        sa.Column(
            "connected_by_user_id", postgresql.UUID(as_uuid=True), nullable=True
        ),
        sa.Column("last_warned_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("connected_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("last_refreshed_at", sa.DateTime(timezone=True), nullable=True),
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
    )
    op.create_index(
        "ix_agency_social_connections_agency_id",
        "agency_social_connections",
        ["agency_id"],
    )
    op.create_index(
        "ix_agency_social_connections_agency_primary",
        "agency_social_connections",
        ["agency_id", "is_primary"],
    )


def downgrade() -> None:
    op.drop_index(
        "ix_agency_social_connections_agency_primary",
        table_name="agency_social_connections",
    )
    op.drop_index(
        "ix_agency_social_connections_agency_id",
        table_name="agency_social_connections",
    )
    op.drop_table("agency_social_connections")
```

- [ ] **Step 2: Confirm single head before upgrading**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/alembic heads`
Expected: a single head line ending in `m1a2b3c4d5e6 (head)`. If two heads appear, fix `down_revision` to the real current head and re-run.

- [ ] **Step 3: Apply to the dev DB**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/alembic upgrade head`
Expected: `Running upgrade z0t1u2v3w4x5 -> m1a2b3c4d5e6`

- [ ] **Step 4: Verify the table exists**

Run: `psql -d sideline_realestate -c "\d agency_social_connections"`
Expected: table description listing all columns above.

- [ ] **Step 5: Commit**

```bash
git add alembic/versions/m1a2b3c4d5e6_add_agency_social_connections.py
git commit -m "feat(meta): migration for agency_social_connections"
```

> **Note for execution:** Apply to all tenant DBs + `slate_template` with `scripts/migrate_all_tenants.py` only at deploy time (per `CLAUDE.md`) — NOT during local development.

---

### Task 4: `MetaGraphClient` (SDK-lite)

**Files:**
- Create: `src/integrations/meta_graph.py`

- [ ] **Step 1: Write the client**

Create `src/integrations/meta_graph.py`:

```python
from typing import Any, Optional

import httpx
import structlog

from src.config import settings

logger = structlog.get_logger()


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

    async def exchange_for_long_lived(self, short_token: str) -> dict[str, Any]:
        return await self._get(
            "/oauth/access_token",
            {
                "grant_type": "fb_exchange_token",
                "client_id": self.app_id,
                "client_secret": self.app_secret,
                "fb_exchange_token": short_token,
            },
        )

    async def refresh_long_lived(self, long_token: str) -> dict[str, Any]:
        return await self.exchange_for_long_lived(long_token)

    async def get_pages(self, user_token: str) -> list[dict[str, Any]]:
        data = await self._get(
            "/me/accounts",
            {"access_token": user_token, "fields": "id,name,access_token"},
        )
        return data.get("data", [])

    async def get_ig_account(
        self, page_id: str, page_token: str
    ) -> Optional[dict[str, Any]]:
        data = await self._get(
            f"/{page_id}",
            {
                "access_token": page_token,
                "fields": "instagram_business_account{id,username}",
            },
        )
        return data.get("instagram_business_account")
```

- [ ] **Step 2: Verify it imports**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.integrations.meta_graph import MetaGraphClient, MetaGraphError; print(MetaGraphClient().base_url)"`
Expected: `https://graph.facebook.com/v21.0`

- [ ] **Step 3: Commit**

```bash
git add src/integrations/meta_graph.py
git commit -m "feat(meta): add MetaGraphClient (Graph API SDK-lite)"
```

---

### Task 5: Backend API — connect / callback / connections / disconnect

**Files:**
- Create: `src/api/meta_social.py`
- Modify: `src/main.py`

- [ ] **Step 1: Write the API module**

Create `src/api/meta_social.py`:

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
    "pages_manage_metadata",
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
Expected: includes `/api/auth/meta/connect`, `/api/auth/meta/connect/instagram`, `/api/auth/meta/callback`, `/api/social/meta/connections`.

- [ ] **Step 4: Commit**

```bash
git add src/api/meta_social.py src/main.py
git commit -m "feat(meta): connect/callback/connections endpoints"
```

---

### Task 6: Frontend types + API client

**Files:**
- Modify: `frontend/src/types/index.ts`
- Create: `frontend/src/api/metaCredentials.ts`

- [ ] **Step 1: Add types**

Append to `frontend/src/types/index.ts`:

```ts
export interface MetaConnection {
  id: string;
  platform: "facebook" | "instagram";
  page_id: string | null;
  ig_account_id: string | null;
  page_name: string | null;
  ig_username: string | null;
  is_primary: boolean;
  status: string;
  token_expires_at: string | null;
  warning_level: number;
}

export interface MetaConnectResponse {
  authorize_url: string;
}
```

- [ ] **Step 2: Add API calls**

Create `frontend/src/api/metaCredentials.ts`:

```ts
import { del, get } from "./client";
import type { MetaConnection, MetaConnectResponse } from "../types";

export const fetchMetaConnections = () =>
  get<MetaConnection[]>("/social/meta/connections");

export const metaConnectFacebook = () =>
  get<MetaConnectResponse>("/auth/meta/connect");

export const metaConnectInstagram = () =>
  get<MetaConnectResponse>("/auth/meta/connect/instagram");

export const metaDisconnect = () => del("/social/meta/connections");
```

- [ ] **Step 3: Verify the frontend type-checks**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate/frontend" && npx tsc --noEmit`
Expected: no errors referencing `metaCredentials.ts` or `MetaConnection`.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/types/index.ts frontend/src/api/metaCredentials.ts
git commit -m "feat(meta): frontend types + api client"
```

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

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/integrations/MetaAccountCard.tsx frontend/src/pages/MetaSettingsPage.tsx frontend/src/pages/IntegrationsPage.tsx
git commit -m "feat(meta): Integrations Meta tab (Facebook + Instagram cards)"
```

---

### Task 8: Manual end-to-end verification (Milestone 1)

**Pre-req:** A real Meta app exists; set `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET`, `PUBLIC_BASE_URL` in `.env`. In the Meta app dashboard, add `{PUBLIC_BASE_URL}/api/auth/meta/callback` to Valid OAuth Redirect URIs.

- [ ] **Step 1: Run backend + frontend**

Backend: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/uvicorn src.main:app --reload`
Frontend: `cd frontend && npm run dev`

- [ ] **Step 2: Connect Facebook**

Navigate to Integrations → Meta. Click **Connect Facebook**. Complete the Facebook OAuth dialog in the popup. The popup should show "Facebook connected successfully" and close; the Facebook card pill flips to **Connected**.

- [ ] **Step 3: Verify rows landed**

Run: `psql -d sideline_realestate -c "SELECT platform, page_name, ig_username, is_primary, status, token_expires_at IS NOT NULL AS has_expiry FROM agency_social_connections;"`
Expected: one `facebook` row per Page (`is_primary=t` on the first), plus an `instagram` row for any Page with a linked IG Business account; `status=connected`, `has_expiry=t`.

- [ ] **Step 4: Verify disconnect**

In the UI (or `curl -X DELETE` with auth), trigger disconnect, then re-run the query. Expected: zero rows. Reconnect to confirm idempotency (no duplicate rows).

---

## Milestone 2 — Instagram after Facebook

The backend already auto-derives IG during the Facebook connect (Task 5 `_store_connection`) and exposes `/api/auth/meta/connect/instagram` (gated on an existing Facebook row). The frontend already disables the Instagram card until Facebook is connected (Task 7). This milestone is verification + the explicit IG-scope re-consent path.

### Task 9: Manual verification of Instagram gating + connect

- [ ] **Step 1: Verify the gate**

With no Facebook connection (after a disconnect), confirm the **Connect Instagram** button is greyed out / disabled and `GET /api/auth/meta/connect/instagram` returns `409 Connect a Facebook Page first`.

Run (with a valid bearer token): `curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" "$PUBLIC_BASE_URL/api/auth/meta/connect/instagram"`
Expected: `409`.

- [ ] **Step 2: Verify connect-after-Facebook**

Connect Facebook first, then click **Connect Instagram**. The popup runs the OAuth with IG scopes and returns success. Confirm the `instagram` row(s) exist and the Instagram pill shows **Connected**:

Run: `psql -d sideline_realestate -c "SELECT platform, ig_username, status FROM agency_social_connections WHERE platform='instagram';"`
Expected: one row per linked IG Business account, `status=connected`.

- [ ] **Step 3: Commit (if any wording/scope tweaks were needed)**

```bash
git add -A && git commit -m "feat(meta): verify Instagram-after-Facebook gating" --allow-empty
```

---

## Milestone 3 — Daily token monitor + silent refresh + warnings (with tests)

### Task 10: `meta_token_monitor` service

**Files:**
- Create: `src/services/meta_token_monitor.py`

- [ ] **Step 1: Write the service**

Create `src/services/meta_token_monitor.py`:

```python
import asyncio
from datetime import datetime, timedelta, timezone

import asyncpg
import structlog
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from src.api.meta_social import _deobfuscate, _obfuscate
from src.config import list_available_tenants
from src.db.session import async_session_factory, get_tenant_session_factory
from src.integrations.meta_graph import MetaGraphClient, MetaGraphError
from src.models.agency_social_connection import AgencySocialConnection
from src.models.user import User

logger = structlog.get_logger()

CHECK_INTERVAL_SECONDS = 24 * 60 * 60
WARN_14 = 14
WARN_5 = 5
RECONNECT_MESSAGE = (
    "Your Meta (Facebook/Instagram) connection in Slate could not be "
    "refreshed automatically and needs to be reconnected. Open "
    "Integrations -> Meta and click Connect Facebook again."
)


async def _set_agency(
    session: AsyncSession, agency_id: str, **values
) -> None:
    await session.execute(
        update(AgencySocialConnection)
        .where(AgencySocialConnection.agency_id == agency_id)
        .values(**values)
    )


async def notify_reconnect(
    session: AsyncSession, agency_id: str, connected_by_user_id
) -> None:
    from src.db.repositories.user import UserRepository
    from src.internal_agents import get_internal_agent

    emails: list[str] = []
    if connected_by_user_id:
        initiator = await session.get(User, connected_by_user_id)
        if initiator and initiator.is_active:
            emails = [initiator.email]
    if not emails:
        repo = UserRepository(session)
        emails = [u.email for u in await repo.get_admins()]
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
                _deobfuscate(primary.user_access_token)
            )
        except MetaGraphError:
            await _set_agency(
                session,
                agency_id,
                status="needs_reconnect",
                last_error="silent refresh failed",
            )
            await notify_reconnect(session, agency_id, primary.connected_by_user_id)
            return
        new_expiry = now + timedelta(seconds=int(refreshed.get("expires_in", 0)))
        await _set_agency(
            session,
            agency_id,
            user_access_token=_obfuscate(refreshed["access_token"]),
            token_expires_at=new_expiry,
            last_refreshed_at=now,
        )
        token_expires_at = new_expiry
        days_remaining = (token_expires_at - now).days

    if days_remaining <= 0:
        await _set_agency(session, agency_id, status="expired")
        return
    if days_remaining <= WARN_5 and primary.warning_level != WARN_5:
        await _set_agency(
            session,
            agency_id,
            warning_level=WARN_5,
            status="expiring",
            last_warned_at=now,
        )
        return
    if days_remaining <= WARN_14 and primary.warning_level == 0:
        await _set_agency(
            session,
            agency_id,
            warning_level=WARN_14,
            status="expiring",
            last_warned_at=now,
        )
        return
    if days_remaining > WARN_14:
        await _set_agency(
            session, agency_id, warning_level=0, status="connected"
        )


async def process_agency_tokens(
    session: AsyncSession, client: MetaGraphClient, now: datetime
) -> None:
    result = await session.execute(
        select(AgencySocialConnection).where(
            AgencySocialConnection.is_primary.is_(True)
        )
    )
    for primary in result.scalars().all():
        await _process_one_agency(session, client, primary, now)
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

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.services.meta_token_monitor import process_agency_tokens, meta_token_monitor_loop; print('ok')"`
Expected: `ok`

- [ ] **Step 3: Commit**

```bash
git add src/services/meta_token_monitor.py
git commit -m "feat(meta): daily token monitor (refresh + warnings)"
```

---

### Task 11: Add `get_admins()` to `UserRepository`

**Files:**
- Modify: `src/db/repositories/user.py`

- [ ] **Step 1: Add the method**

Add to the `UserRepository` class (with `from src.models.user import User` and `from sqlalchemy import select` already imported, or add them):

```python
    async def get_admins(self) -> list[User]:
        result = await self.session.execute(
            select(User).where(
                User.role.in_(["owner", "admin"]), User.is_active.is_(True)
            )
        )
        return list(result.scalars().all())
```

- [ ] **Step 2: Verify import**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.db.repositories.user import UserRepository; print(hasattr(UserRepository, 'get_admins'))"`
Expected: `True`

- [ ] **Step 3: Commit**

```bash
git add src/db/repositories/user.py
git commit -m "feat(meta): UserRepository.get_admins for reconnect notification"
```

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

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.main import app; print('ok')"`
Expected: `ok`

- [ ] **Step 4: Commit**

```bash
git add src/main.py
git commit -m "feat(meta): start token monitor loop in lifespan"
```

---

### Task 13: Integration test — refresh attempted + success resets warning

**Files:**
- Create: `tests/integration/test_meta_token_monitor.py`

- [ ] **Step 1: Write the failing test**

Create `tests/integration/test_meta_token_monitor.py`:

```python
from datetime import datetime, timedelta, timezone

import pytest
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.api.meta_social import _deobfuscate, _obfuscate
from src.models.agency_social_connection import AgencySocialConnection
from src.services import meta_token_monitor
from src.services.meta_token_monitor import process_agency_tokens


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
            platform="facebook",
            page_id="PAGE1",
            page_name="Page One",
            access_token=_obfuscate("page-token"),
            is_primary=True,
            user_access_token=_obfuscate("old-user-token"),
            token_expires_at=expires,
            warning_level=0,
            status="connected",
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
    assert _deobfuscate(row.user_access_token) == "fresh-user-token"
    assert row.warning_level == 0
    assert row.status == "connected"
    assert (row.token_expires_at - now).days > WARN_14_DAYS


WARN_14_DAYS = 14
```

- [ ] **Step 2: Run it to verify it fails (table/logic wired)**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_token_monitor.py::test_refresh_success_resets_state -v`
Expected: PASS once the service from Task 10 is correct. If it FAILS, read the assertion and fix `meta_token_monitor.py` (this is the implementation-verifying test for the success path).

- [ ] **Step 3: Commit**

```bash
git add tests/integration/test_meta_token_monitor.py
git commit -m "test(meta): monitor refresh-success resets warning state"
```

---

### Task 14: Integration test — hard refresh failure ⇒ needs_reconnect + notify once

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
    assert row.status == "needs_reconnect"
    assert calls["n"] == 1
```

> Note: dedupe-to-once works because after the first tick `status="needs_reconnect"`; the second tick still attempts refresh (still ≤14 days) and fails again, so to guarantee "once" the implementation must skip notifying when `status` is already `needs_reconnect`. Add that guard in `_process_one_agency`'s hard-failure branch: only call `notify_reconnect` if `primary.status != "needs_reconnect"`. Update `src/services/meta_token_monitor.py` accordingly, then re-run.

- [ ] **Step 2: Add the dedupe guard to the service**

In `_process_one_agency`, change the `except MetaGraphError:` branch to:

```python
        except MetaGraphError:
            if primary.status != "needs_reconnect":
                await notify_reconnect(
                    session, agency_id, primary.connected_by_user_id
                )
            await _set_agency(
                session,
                agency_id,
                status="needs_reconnect",
                last_error="silent refresh failed",
            )
            return
```

- [ ] **Step 3: Run the test**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_token_monitor.py::test_refresh_failure_sets_needs_reconnect_and_notifies_once -v`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add tests/integration/test_meta_token_monitor.py src/services/meta_token_monitor.py
git commit -m "test(meta): refresh-failure -> needs_reconnect + notify once"
```

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
    assert row.warning_level == 14
    assert row.status == "expiring"

    row.token_expires_at = now + timedelta(days=4)
    await db_session.flush()
    client2 = NoExtendGraphClient(days_left=4, now=now)
    await process_agency_tokens(db_session, client2, now)
    await db_session.refresh(row)
    assert row.warning_level == 5

    row.token_expires_at = now - timedelta(days=1)
    await db_session.flush()
    client3 = NoExtendGraphClient(days_left=-1, now=now)
    await process_agency_tokens(db_session, client3, now)
    await db_session.refresh(row)
    assert row.status == "expired"


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
    assert row.status == "connected"
```

- [ ] **Step 2: Run the tests**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS. Fix `meta_token_monitor.py` threshold logic if any fail.

- [ ] **Step 3: Commit**

```bash
git add tests/integration/test_meta_token_monitor.py
git commit -m "test(meta): warning thresholds + healthy no-op"
```

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
    assert by_agency["agency-a"].status == "connected"
    assert by_agency["agency-a"].warning_level == 0
    assert by_agency["agency-b"].status in ("expiring", "connected")
```

- [ ] **Step 2: Run the full job test file**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS.

- [ ] **Step 3: Commit**

```bash
git add tests/integration/test_meta_token_monitor.py
git commit -m "test(meta): per-agency isolation in monitor"
```

---

## Milestone 4 — Reconnect notification verification

The notification path is implemented in Task 10/Step 2 and covered by the Task 14 test. This milestone is a manual smoke check of the actual `send_notification` wiring.

### Task 17: Manual smoke test of the reconnect notification

- [ ] **Step 1: Seed a near-failure agency in the dev DB**

Run:
```bash
psql -d sideline_realestate -c "UPDATE agency_social_connections SET token_expires_at = now() + interval '3 days', user_access_token = 'aW52YWxpZA==' WHERE is_primary = true;"
```
(`aW52YWxpZA==` is base64 for `invalid`, so the refresh call will fail against Meta.)

- [ ] **Step 2: Trigger one monitor pass manually**

Run:
```bash
cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "
import asyncio
from datetime import datetime, timezone
from src.db.session import async_session_factory
from src.integrations.meta_graph import MetaGraphClient
from src.services.meta_token_monitor import process_agency_tokens

async def main():
    async with async_session_factory() as db:
        await process_agency_tokens(db, MetaGraphClient(), datetime.now(timezone.utc))
        await db.commit()

asyncio.run(main())
"
```
Expected: log line `meta_graph.error` (refresh failed) and the row status becomes `needs_reconnect`. With at least one active owner/admin user present, `send_notification` is invoked (in sim/log mode it logs the send).

- [ ] **Step 3: Verify status**

Run: `psql -d sideline_realestate -c "SELECT status, last_error FROM agency_social_connections WHERE is_primary = true;"`
Expected: `status=needs_reconnect`, `last_error='silent refresh failed'`.

- [ ] **Step 4: Reset the dev row (reconnect via UI) and commit any fixes**

```bash
git add -A && git commit -m "chore(meta): verify reconnect notification path" --allow-empty
```

---

## Final verification

- [ ] **Run the full job test suite**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS.

- [ ] **Backend boots; frontend builds**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.main import app; print('ok')" && cd frontend && npx tsc --noEmit && npm run build`
Expected: `ok` + clean type-check + successful build.

- [ ] **Deploy-time only:** run `scripts/migrate_all_tenants.py` to apply the migration to every tenant + `slate_template` (per `CLAUDE.md`). Do NOT run during local dev.
