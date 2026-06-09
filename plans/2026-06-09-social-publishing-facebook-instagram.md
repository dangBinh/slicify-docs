# Social Publishing (Facebook / Instagram) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add immediate (`POST /api/posts/publish`) and scheduled (`POST /api/posts/schedule`) publishing of image posts to an agency's connected Facebook Pages and Instagram Business accounts, with per-target results, auto-retry, an append-only attempt log, and actionable errors for expired tokens, rate limits, and bad media.

**Architecture:** A fan-out model — one `Post` → many `PostTarget`s (one per connected account). Both endpoints create the same rows; a single `SocialPublishService` executor does the Graph API calls, error mapping, and attempt logging. Immediate publish runs the executor inline; scheduled publish is fired by a background `social_post_poller_loop` that iterates every tenant DB (mirroring the existing `meta_token_monitor_loop`). Media is uploaded into the existing storage service and resolved to a public URL at publish time.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2 (async), Pydantic v2, Alembic, Playwright-unrelated, `httpx` (Meta Graph client), Pillow (image validation), pytest + pytest-asyncio, structlog. Builds on DEV-170 (Meta OAuth: `agency_social_connections` / `agency_social_accounts`, Fernet `encrypt`/`decrypt`, `MetaGraphClient`).

---

## Spec

Source spec: `slicify-docs/specs/2026-06-09-social-publishing-facebook-instagram-design.md`. Read it before starting.

## Conventions (from `src/CLAUDE.md` — non-negotiable)

- **No comments / docstrings in code.** All code below is comment-free on purpose — keep it that way.
- Layers: router (`src/api/`) → service (`src/services/`) → repository (`src/db/repositories/`) → model (`src/models/`). Schemas in `src/schemas/`. Routers contain no business logic.
- Services raise `ValueError` for domain errors; never `HTTPException`. Endpoints catch `ValueError` → `HTTPException(400)`.
- Authenticated endpoints use `Depends(get_tenant_scoped_db)`.
- Functions < 20 lines preferred, ≤ 100 max; max 1 nesting level; early returns.
- `structlog.get_logger()` only. Named constants for any repeated literal.

## Testing scope (limited to critical paths + core functionality)

Integration tests cover **only** the critical paths and core behaviors, not exhaustive permutations:

- Executor success path — Facebook single image **and** Instagram container flow.
- The three required error behaviors — expired token → `needs_reconnect`, rate limit → retry with reset, media rejection mapping.
- Attempt logging is asserted within the expired-token executor test.
- Scheduled-post firing (poller publishes a due post).
- The two highest-value API guards — Instagram-requires-media and unauthorized-account → reconnect `400`.

Everything else (URL-resolver fallback/none cases, default rate-limit reset, expired/unknown error-mapping unit cases, retry-due poller wiring, basic 422s) is left to manual/code-review verification. **~12 integration tests total.**

## Environment

- Backend root: `/Users/binhquach/Workplace/Slicify AI/slicify-realestate`
- Python: `venv/bin/python`, tests: `venv/bin/pytest`, alembic: `venv/bin/alembic`
- A local Postgres `sideline_realestate` (dev) and `sideline_realestate_test` (tests) must exist. Tests build all tables from `Base.metadata` each run, so the test DB only needs to exist.

## File Structure (created / modified)

**Create**
- `src/models/post.py` — `Post`
- `src/models/post_target.py` — `PostTarget`
- `src/models/media.py` — `Media`
- `src/models/post_publish_attempt.py` — `PostPublishAttempt`
- `src/db/repositories/post.py` — `PostRepository`
- `src/db/repositories/post_target.py` — `PostTargetRepository`
- `src/db/repositories/media.py` — `MediaRepository`
- `src/db/repositories/post_publish_attempt.py` — `PostPublishAttemptRepository`
- `src/db/repositories/agency_social_account.py` — `AgencySocialAccountRepository`
- `src/schemas/posts.py` — request/response schemas + `SocialPublishValidationError`
- `src/services/meta_error_mapping.py` — `classify_meta_error`, `PublishErrorOutcome`
- `src/services/social_publish_service.py` — `SocialPublishService` (executor)
- `src/services/social_post_poller.py` — `SocialPostRunner`, `social_post_poller_loop`
- `src/api/posts.py` — routers
- `alembic/versions/<auto>_add_social_posts.py` — migration
- `tests/integration/test_meta_error_mapping.py`
- `tests/integration/test_social_publish_service.py`
- `tests/integration/test_social_post_poller.py`
- `tests/integration/test_social_media_url.py`
- `tests/integration/test_api_posts.py`

**Modify**
- `src/constants.py` — `PostStatus`, `PostTargetStatus`, `PublishErrorCode`, Meta error-code constants
- `src/config.py` — social config keys
- `src/models/__init__.py` — register the 4 new models
- `src/integrations/meta_graph.py` — `_post()`, richer `MetaGraphError`, publish methods
- `src/services/storage.py` — `upload_file(..., content_type=...)`
- `src/utils.py` — `resolve_social_public_media_url`, `SOCIAL_*` image helpers
- `src/auth/permissions.py` — `posts:publish`
- `src/main.py` — register posts router + start poller loop
- `tests/fakes/meta_graph.py` — `FakePublishGraphClient`
- `tests/factories/social.py` — `seed_connected_account`

---

# PHASE 0 — Data + config foundation

## Task 1: Constants & enums

**Files:**
- Modify: `src/constants.py` (append after the existing `MetaConnectionStatus` enum)

- [ ] **Step 1: Add enums and Meta error-code constants**

Append to `src/constants.py`:

```python
class PostStatus(str, Enum):
    SCHEDULED = "scheduled"
    PUBLISHING = "publishing"
    PUBLISHED = "published"
    PARTIALLY_FAILED = "partially_failed"
    FAILED = "failed"
    CANCELLED = "cancelled"


class PostTargetStatus(str, Enum):
    PENDING = "pending"
    PUBLISHING = "publishing"
    PUBLISHED = "published"
    FAILED = "failed"
    RATE_LIMITED = "rate_limited"
    NEEDS_RECONNECT = "needs_reconnect"


class PublishErrorCode(str, Enum):
    EXPIRED_TOKEN = "expired_token"
    RATE_LIMITED = "rate_limited"
    MEDIA_INVALID = "media_invalid"
    TRANSIENT = "transient"
    UNKNOWN = "unknown"


class MediaType(str, Enum):
    IMAGE = "image"


META_OAUTH_ERROR_CODES = frozenset({190})
META_OAUTH_ERROR_SUBCODES = frozenset({458, 460, 463, 467, 492})
META_RATE_LIMIT_ERROR_CODES = frozenset({4, 17, 32, 613, 80007})
META_MEDIA_INVALID_ERROR_CODES = frozenset({9004, 36000, 36003})
META_MEDIA_INVALID_ERROR_SUBCODES = frozenset(
    {2207003, 2207004, 2207006, 2207020, 2207026, 2207032}
)
META_TRANSIENT_ERROR_CODES = frozenset({1, 2})
```

- [ ] **Step 2: Verify it imports**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.constants import PostStatus, PostTargetStatus, PublishErrorCode, MediaType, META_RATE_LIMIT_ERROR_CODES; print(PostStatus.PUBLISHED.value, sorted(META_RATE_LIMIT_ERROR_CODES))"`
Expected: `published [4, 17, 32, 613, 80007]`

- [ ] **Step 3: Commit**

```bash
git add src/constants.py
git commit -m "feat(social): add post/publish enums and Meta error-code constants"
```

---

## Task 2: Config keys

**Files:**
- Modify: `src/config.py` (add fields inside `class Settings`, near the existing `META_*` block around line 51)

- [ ] **Step 1: Add settings**

Add these lines to `class Settings` in `src/config.py`, immediately after `META_TOKEN_REFRESH_THRESHOLD_DAYS`:

```python
    LOCAL_MEDIA_BASE_URL: str = ""
    SOCIAL_POST_POLL_INTERVAL_SECONDS: int = 60
    SOCIAL_POST_MAX_ATTEMPTS: int = 5
    SOCIAL_POST_RETRY_BASE_SECONDS: int = 60
    SOCIAL_POST_RETRY_MAX_BACKOFF_SECONDS: int = 6 * 60 * 60
    SOCIAL_MAX_IMAGE_BYTES: int = 8 * 1024 * 1024
    SOCIAL_MAX_CAROUSEL_ITEMS: int = 10
    SOCIAL_MEDIA_ORPHAN_TTL_HOURS: int = 6
    SOCIAL_IG_STATUS_POLL_ATTEMPTS: int = 5
    SOCIAL_IG_STATUS_POLL_INTERVAL_SECONDS: float = 2.0
    IG_CONTENT_PUBLISH_LIMIT_PER_24H: int = 100
```

- [ ] **Step 2: Verify defaults load**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.config import settings; print(settings.SOCIAL_POST_MAX_ATTEMPTS, settings.SOCIAL_MAX_IMAGE_BYTES, repr(settings.LOCAL_MEDIA_BASE_URL))"`
Expected: `5 8388608 ''`

- [ ] **Step 3: Add to `.env.template`**

Append to `.env.template` (so new tenants inherit it):

```
# Social publishing — local dev media tunnel (e.g. ngrok) so Instagram can fetch local images
LOCAL_MEDIA_BASE_URL=
```

- [ ] **Step 4: Commit**

```bash
git add src/config.py .env.template
git commit -m "feat(social): add social publishing config keys"
```

---

## Task 3: Models

**Files:**
- Create: `src/models/post.py`, `src/models/post_target.py`, `src/models/media.py`, `src/models/post_publish_attempt.py`
- Modify: `src/models/__init__.py`

- [ ] **Step 1: Create `src/models/post.py`**

```python
import uuid
from datetime import datetime
from typing import TYPE_CHECKING, Optional

from sqlalchemy import Boolean, DateTime, ForeignKey, String, Text
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from src.models.base import Base, TimestampMixin, UUIDMixin

if TYPE_CHECKING:
    from src.models.media import Media
    from src.models.post_target import PostTarget


class Post(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "posts"

    agency_id: Mapped[str] = mapped_column(String(100), nullable=False, index=True)
    content: Mapped[Optional[str]] = mapped_column(Text)
    status: Mapped[str] = mapped_column(
        String(20), nullable=False, server_default="scheduled"
    )
    scheduled_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    published_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    needs_attention: Mapped[bool] = mapped_column(
        Boolean, nullable=False, server_default="false"
    )
    created_by_user_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )

    targets: Mapped[list["PostTarget"]] = relationship(
        "PostTarget",
        back_populates="post",
        cascade="all, delete-orphan",
        lazy="selectin",
    )
    media: Mapped[list["Media"]] = relationship(
        "Media",
        back_populates="post",
        cascade="all, delete-orphan",
        order_by="Media.position",
        lazy="selectin",
    )
```

- [ ] **Step 2: Create `src/models/post_target.py`**

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
)
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from src.models.base import Base, TimestampMixin, UUIDMixin

if TYPE_CHECKING:
    from src.models.post import Post


class PostTarget(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "post_targets"

    post_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("posts.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    social_account_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("agency_social_accounts.id"),
        nullable=False,
    )
    platform: Mapped[str] = mapped_column(String(20), nullable=False)
    status: Mapped[str] = mapped_column(
        String(20), nullable=False, server_default="pending"
    )
    provider_post_id: Mapped[Optional[str]] = mapped_column(String(100))
    provider_container_id: Mapped[Optional[str]] = mapped_column(String(100))
    attempt_count: Mapped[int] = mapped_column(
        SmallInteger, nullable=False, server_default="0"
    )
    next_attempt_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True)
    )
    rate_limit_reset_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True)
    )
    last_error_code: Mapped[Optional[str]] = mapped_column(String(50))
    last_error_detail: Mapped[Optional[str]] = mapped_column(Text)
    published_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))

    post: Mapped["Post"] = relationship("Post", back_populates="targets")
```

- [ ] **Step 3: Create `src/models/media.py`**

```python
import uuid
from typing import TYPE_CHECKING, Optional

from sqlalchemy import ForeignKey, Integer, SmallInteger, String
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from src.models.base import Base, TimestampMixin, UUIDMixin

if TYPE_CHECKING:
    from src.models.post import Post


class Media(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "media"

    post_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("posts.id", ondelete="CASCADE"),
        index=True,
    )
    agency_id: Mapped[str] = mapped_column(String(100), nullable=False, index=True)
    storage_key: Mapped[str] = mapped_column(String(500), nullable=False)
    public_url: Mapped[Optional[str]] = mapped_column(String(1000))
    media_type: Mapped[str] = mapped_column(
        String(20), nullable=False, server_default="image"
    )
    content_type: Mapped[str] = mapped_column(String(100), nullable=False)
    width: Mapped[Optional[int]] = mapped_column(Integer)
    height: Mapped[Optional[int]] = mapped_column(Integer)
    byte_size: Mapped[int] = mapped_column(Integer, nullable=False)
    position: Mapped[int] = mapped_column(
        SmallInteger, nullable=False, server_default="0"
    )

    post: Mapped[Optional["Post"]] = relationship("Post", back_populates="media")
```

- [ ] **Step 4: Create `src/models/post_publish_attempt.py`**

```python
import uuid
from datetime import datetime
from typing import Optional

from sqlalchemy import (
    JSON,
    DateTime,
    ForeignKey,
    Integer,
    SmallInteger,
    String,
    Text,
    func,
)
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from src.models.base import Base, UUIDMixin


class PostPublishAttempt(Base, UUIDMixin):
    __tablename__ = "post_publish_attempts"

    post_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("posts.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    post_target_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("post_targets.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    platform: Mapped[str] = mapped_column(String(20), nullable=False)
    account_provider_id: Mapped[str] = mapped_column(String(100), nullable=False)
    attempted_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.clock_timestamp(), nullable=False
    )
    outcome: Mapped[str] = mapped_column(String(20), nullable=False)
    provider_post_id: Mapped[Optional[str]] = mapped_column(String(100))
    error_code: Mapped[Optional[str]] = mapped_column(String(50))
    error_detail: Mapped[Optional[str]] = mapped_column(Text)
    http_status: Mapped[Optional[int]] = mapped_column(SmallInteger)
    raw_response: Mapped[Optional[dict]] = mapped_column(JSON)
    duration_ms: Mapped[Optional[int]] = mapped_column(Integer)
    simulated: Mapped[bool] = mapped_column(
        nullable=False, server_default="false"
    )
```

- [ ] **Step 5: Register models in `src/models/__init__.py`**

Add these imports next to the existing `agency_social_*` imports:

```python
from src.models.media import Media
from src.models.post import Post
from src.models.post_publish_attempt import PostPublishAttempt
from src.models.post_target import PostTarget
```

And add to the `__all__` list:

```python
    "Media",
    "Post",
    "PostPublishAttempt",
    "PostTarget",
```

- [ ] **Step 6: Verify the metadata builds**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.models import Base, Post, PostTarget, Media, PostPublishAttempt; print(sorted(t for t in Base.metadata.tables if t in {'posts','post_targets','media','post_publish_attempts'}))"`
Expected: `['media', 'post_publish_attempts', 'post_targets', 'posts']`

- [ ] **Step 7: Commit**

```bash
git add src/models/post.py src/models/post_target.py src/models/media.py src/models/post_publish_attempt.py src/models/__init__.py
git commit -m "feat(social): add Post, PostTarget, Media, PostPublishAttempt models"
```

---

## Task 4: Alembic migration

**Files:**
- Create: `alembic/versions/<auto>_add_social_posts.py` (scaffolded by alembic)

- [ ] **Step 1: Scaffold the migration (auto-fills revision + down_revision)**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/alembic revision -m "add_social_posts"`
Expected: prints `Generating .../alembic/versions/<id>_add_social_posts.py ... done`. Open that file. Leave the auto-generated `revision` and `down_revision` lines untouched.

- [ ] **Step 2: Fill `upgrade()` and `downgrade()`**

Replace the body of the generated file's `upgrade()` / `downgrade()` with:

```python
import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql


def upgrade() -> None:
    op.create_table(
        "posts",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("agency_id", sa.String(length=100), nullable=False),
        sa.Column("content", sa.Text(), nullable=True),
        sa.Column("status", sa.String(length=20), server_default="scheduled", nullable=False),
        sa.Column("scheduled_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("published_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("needs_attention", sa.Boolean(), server_default="false", nullable=False),
        sa.Column("created_by_user_id", postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.ForeignKeyConstraint(["created_by_user_id"], ["users.id"]),
    )
    op.create_index("ix_posts_agency_id", "posts", ["agency_id"])

    op.create_table(
        "media",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("post_id", postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column("agency_id", sa.String(length=100), nullable=False),
        sa.Column("storage_key", sa.String(length=500), nullable=False),
        sa.Column("public_url", sa.String(length=1000), nullable=True),
        sa.Column("media_type", sa.String(length=20), server_default="image", nullable=False),
        sa.Column("content_type", sa.String(length=100), nullable=False),
        sa.Column("width", sa.Integer(), nullable=True),
        sa.Column("height", sa.Integer(), nullable=True),
        sa.Column("byte_size", sa.Integer(), nullable=False),
        sa.Column("position", sa.SmallInteger(), server_default="0", nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.ForeignKeyConstraint(["post_id"], ["posts.id"], ondelete="CASCADE"),
    )
    op.create_index("ix_media_post_id", "media", ["post_id"])
    op.create_index("ix_media_agency_id", "media", ["agency_id"])

    op.create_table(
        "post_targets",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("post_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("social_account_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("platform", sa.String(length=20), nullable=False),
        sa.Column("status", sa.String(length=20), server_default="pending", nullable=False),
        sa.Column("provider_post_id", sa.String(length=100), nullable=True),
        sa.Column("provider_container_id", sa.String(length=100), nullable=True),
        sa.Column("attempt_count", sa.SmallInteger(), server_default="0", nullable=False),
        sa.Column("next_attempt_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("rate_limit_reset_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("last_error_code", sa.String(length=50), nullable=True),
        sa.Column("last_error_detail", sa.Text(), nullable=True),
        sa.Column("published_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.ForeignKeyConstraint(["post_id"], ["posts.id"], ondelete="CASCADE"),
        sa.ForeignKeyConstraint(["social_account_id"], ["agency_social_accounts.id"]),
    )
    op.create_index("ix_post_targets_post_id", "post_targets", ["post_id"])

    op.create_table(
        "post_publish_attempts",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("post_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("post_target_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("platform", sa.String(length=20), nullable=False),
        sa.Column("account_provider_id", sa.String(length=100), nullable=False),
        sa.Column("attempted_at", sa.DateTime(timezone=True), server_default=sa.func.clock_timestamp(), nullable=False),
        sa.Column("outcome", sa.String(length=20), nullable=False),
        sa.Column("provider_post_id", sa.String(length=100), nullable=True),
        sa.Column("error_code", sa.String(length=50), nullable=True),
        sa.Column("error_detail", sa.Text(), nullable=True),
        sa.Column("http_status", sa.SmallInteger(), nullable=True),
        sa.Column("raw_response", sa.JSON(), nullable=True),
        sa.Column("duration_ms", sa.Integer(), nullable=True),
        sa.Column("simulated", sa.Boolean(), server_default="false", nullable=False),
        sa.ForeignKeyConstraint(["post_id"], ["posts.id"], ondelete="CASCADE"),
        sa.ForeignKeyConstraint(["post_target_id"], ["post_targets.id"], ondelete="CASCADE"),
    )
    op.create_index("ix_post_publish_attempts_post_id", "post_publish_attempts", ["post_id"])
    op.create_index("ix_post_publish_attempts_post_target_id", "post_publish_attempts", ["post_target_id"])


def downgrade() -> None:
    op.drop_table("post_publish_attempts")
    op.drop_table("post_targets")
    op.drop_table("media")
    op.drop_table("posts")
```

Keep the alembic-generated imports if they differ; the `import sqlalchemy as sa`, `from alembic import op`, and `from sqlalchemy.dialects import postgresql` lines must all be present.

- [ ] **Step 3: Apply to the dev DB**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/alembic upgrade head`
Expected: `Running upgrade <prev> -> <new>, add_social_posts`

- [ ] **Step 4: Verify tables exist**

Run: `psql -d sideline_realestate -c "\dt posts post_targets media post_publish_attempts"`
Expected: all four tables listed.

- [ ] **Step 5: Verify downgrade is clean, then re-upgrade**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/alembic downgrade -1 && venv/bin/alembic upgrade head`
Expected: downgrade then upgrade both succeed with no error.

- [ ] **Step 6: Commit**

```bash
git add alembic/versions/
git commit -m "feat(social): migration for posts, post_targets, media, post_publish_attempts"
```

> **Multi-tenant rollout (deploy-time, not now):** before this ships to UAT/prod, run `SLICIFY_CONTROL_DATABASE_URL=$CONTROL_DB TEMPLATE_DB_URL=$TEMPLATE_DB venv/bin/python scripts/migrate_all_tenants.py` per root `CLAUDE.md`. Covered again in Task 15.

---

## Task 5: Repositories

**Files:**
- Create: `src/db/repositories/post.py`, `src/db/repositories/post_target.py`, `src/db/repositories/media.py`, `src/db/repositories/post_publish_attempt.py`, `src/db/repositories/agency_social_account.py`

- [ ] **Step 1: Create `src/db/repositories/agency_social_account.py`**

```python
import uuid
from typing import Optional, Sequence

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.db.repositories.base import BaseRepository
from src.models.agency_social_account import AgencySocialAccount


class AgencySocialAccountRepository(BaseRepository[AgencySocialAccount]):
    def __init__(self, session: AsyncSession):
        super().__init__(AgencySocialAccount, session)

    async def list_by_ids(
        self, account_ids: Sequence[uuid.UUID]
    ) -> list[AgencySocialAccount]:
        if not account_ids:
            return []
        stmt = select(AgencySocialAccount).where(
            AgencySocialAccount.id.in_(account_ids)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def get_for_agency(
        self, account_id: uuid.UUID, agency_id: str
    ) -> Optional[AgencySocialAccount]:
        stmt = select(AgencySocialAccount).where(
            AgencySocialAccount.id == account_id,
            AgencySocialAccount.agency_id == agency_id,
        )
        return await self.session.scalar(stmt)
```

- [ ] **Step 2: Create `src/db/repositories/post.py`**

```python
import uuid
from datetime import datetime
from typing import Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.constants import PostStatus
from src.db.repositories.base import BaseRepository
from src.models.post import Post


class PostRepository(BaseRepository[Post]):
    def __init__(self, session: AsyncSession):
        super().__init__(Post, session)

    async def list_for_agency(
        self, agency_id: str, offset: int = 0, limit: int = 50
    ) -> list[Post]:
        stmt = (
            select(Post)
            .where(Post.agency_id == agency_id)
            .order_by(Post.created_at.desc())
            .offset(offset)
            .limit(limit)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def get_for_agency(
        self, post_id: uuid.UUID, agency_id: str
    ) -> Optional[Post]:
        stmt = select(Post).where(Post.id == post_id, Post.agency_id == agency_id)
        return await self.session.scalar(stmt)

    async def list_due_scheduled(
        self, now: datetime, limit: int = 100
    ) -> list[Post]:
        stmt = (
            select(Post)
            .where(
                Post.status == PostStatus.SCHEDULED.value,
                Post.scheduled_at.is_not(None),
                Post.scheduled_at <= now,
            )
            .order_by(Post.scheduled_at)
            .limit(limit)
            .with_for_update(skip_locked=True)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

- [ ] **Step 3: Create `src/db/repositories/post_target.py`**

```python
import uuid
from datetime import datetime

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.constants import PostTargetStatus
from src.db.repositories.base import BaseRepository
from src.models.post_target import PostTarget

RETRYABLE_STATUSES = (
    PostTargetStatus.RATE_LIMITED.value,
    PostTargetStatus.FAILED.value,
)


class PostTargetRepository(BaseRepository[PostTarget]):
    def __init__(self, session: AsyncSession):
        super().__init__(PostTarget, session)

    async def list_for_post(self, post_id: uuid.UUID) -> list[PostTarget]:
        stmt = select(PostTarget).where(PostTarget.post_id == post_id)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def list_retry_due(
        self, now: datetime, max_attempts: int, limit: int = 100
    ) -> list[PostTarget]:
        stmt = (
            select(PostTarget)
            .where(
                PostTarget.status.in_(RETRYABLE_STATUSES),
                PostTarget.next_attempt_at.is_not(None),
                PostTarget.next_attempt_at <= now,
                PostTarget.attempt_count < max_attempts,
            )
            .order_by(PostTarget.next_attempt_at)
            .limit(limit)
            .with_for_update(skip_locked=True)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

- [ ] **Step 4: Create `src/db/repositories/media.py`**

```python
import uuid
from datetime import datetime
from typing import Sequence

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.db.repositories.base import BaseRepository
from src.models.media import Media


class MediaRepository(BaseRepository[Media]):
    def __init__(self, session: AsyncSession):
        super().__init__(Media, session)

    async def list_unattached_for_agency(
        self, media_ids: Sequence[uuid.UUID], agency_id: str
    ) -> list[Media]:
        if not media_ids:
            return []
        stmt = select(Media).where(
            Media.id.in_(media_ids),
            Media.agency_id == agency_id,
            Media.post_id.is_(None),
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def list_orphans(self, cutoff: datetime, limit: int = 100) -> list[Media]:
        stmt = (
            select(Media)
            .where(Media.post_id.is_(None), Media.created_at < cutoff)
            .limit(limit)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

- [ ] **Step 5: Create `src/db/repositories/post_publish_attempt.py`**

```python
import uuid

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.db.repositories.base import BaseRepository
from src.models.post_publish_attempt import PostPublishAttempt


class PostPublishAttemptRepository(BaseRepository[PostPublishAttempt]):
    def __init__(self, session: AsyncSession):
        super().__init__(PostPublishAttempt, session)

    async def list_for_post(
        self, post_id: uuid.UUID
    ) -> list[PostPublishAttempt]:
        stmt = (
            select(PostPublishAttempt)
            .where(PostPublishAttempt.post_id == post_id)
            .order_by(PostPublishAttempt.attempted_at)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

- [ ] **Step 6: Verify imports**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.db.repositories.post import PostRepository; from src.db.repositories.post_target import PostTargetRepository; from src.db.repositories.media import MediaRepository; from src.db.repositories.post_publish_attempt import PostPublishAttemptRepository; from src.db.repositories.agency_social_account import AgencySocialAccountRepository; print('ok')"`
Expected: `ok`

- [ ] **Step 7: Commit**

```bash
git add src/db/repositories/post.py src/db/repositories/post_target.py src/db/repositories/media.py src/db/repositories/post_publish_attempt.py src/db/repositories/agency_social_account.py
git commit -m "feat(social): repositories for posts, targets, media, attempts, accounts"
```

---

# PHASE 1 — Integration + plumbing

## Task 6: Storage `content_type` parameter (Spaces-safe)

**Files:**
- Modify: `src/services/storage.py`

> Backward-compatible signature change — existing callers keep the default content type, so the existing suite already covers it. No new test here (limited-testing scope).

- [ ] **Step 1: Add the parameter to both services**

In `src/services/storage.py`, change `LocalStorageService.upload_file`:

```python
    def upload_file(
        self, file_bytes: bytes, key: str, content_type: str = "application/pdf"
    ) -> str:
        dest = _STORAGE_ROOT / key
        dest.parent.mkdir(parents=True, exist_ok=True)
        dest.write_bytes(file_bytes)
        logger.debug("storage.local.upload", key=key, bytes=len(file_bytes))
        return key
```

And `SpacesStorageService.upload_file`:

```python
    def upload_file(
        self, file_bytes: bytes, key: str, content_type: str = "application/pdf"
    ) -> str:
        self._client.put_object(
            Bucket=self._bucket,
            Key=key,
            Body=file_bytes,
            ContentType=content_type,
        )
        url = self._public_url(key)
        logger.info("storage.spaces.upload", key=key, bytes=len(file_bytes), url=url)
        return url
```

- [ ] **Step 2: Verify the existing suite still passes (no regression)**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_api_contacts.py -q`
Expected: PASS — existing callers use the default content type, unchanged.

- [ ] **Step 3: Commit**

```bash
git add src/services/storage.py
git commit -m "feat(social): parameterize storage upload content type (default unchanged)"
```

---

## Task 7: Social media public-URL resolver

**Files:**
- Modify: `src/utils.py` (append `resolve_social_public_media_url` + image constants)
- Test: `tests/integration/test_social_media_url.py`

- [ ] **Step 1: Write the failing tests** (create `tests/integration/test_social_media_url.py`)

These two cover the user-required guarantees: Spaces URLs are never rewritten, and `LOCAL_MEDIA_BASE_URL` takes exclusive precedence over `PUBLIC_BASE_URL`.

```python
from src.utils import resolve_social_public_media_url


def test_resolver_returns_absolute_url_as_is(monkeypatch):
    import src.utils as utils_module

    monkeypatch.setattr(utils_module.settings, "LOCAL_MEDIA_BASE_URL", "https://ngrok.test")
    monkeypatch.setattr(utils_module.settings, "PUBLIC_BASE_URL", "https://app.test")
    url = resolve_social_public_media_url(
        "posts/x.jpg", "https://bucket.sfo3.digitaloceanspaces.com/posts/x.jpg"
    )
    assert url == "https://bucket.sfo3.digitaloceanspaces.com/posts/x.jpg"


def test_resolver_prefers_local_media_base_url_over_public(monkeypatch):
    import src.utils as utils_module

    monkeypatch.setattr(utils_module.settings, "LOCAL_MEDIA_BASE_URL", "https://ngrok.test/")
    monkeypatch.setattr(utils_module.settings, "PUBLIC_BASE_URL", "https://app.test")
    url = resolve_social_public_media_url("posts/x.jpg", None)
    assert url == "https://ngrok.test/storage/posts/x.jpg"
```

- [ ] **Step 2: Run — expect failure**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_media_url.py -v -k resolver`
Expected: FAIL — `ImportError: cannot import name 'resolve_social_public_media_url'`

- [ ] **Step 3: Implement in `src/utils.py`** (append at end of file)

```python
SOCIAL_ALLOWED_IMAGE_TYPES = frozenset({"image/jpeg", "image/png"})
SOCIAL_IG_MIN_ASPECT_RATIO = 0.8
SOCIAL_IG_MAX_ASPECT_RATIO = 1.91


def resolve_social_public_media_url(
    storage_key: str, public_url: Optional[str]
) -> Optional[str]:
    if public_url and public_url.startswith("http"):
        return public_url
    if settings.LOCAL_MEDIA_BASE_URL:
        return f"{settings.LOCAL_MEDIA_BASE_URL.rstrip('/')}/storage/{storage_key}"
    if settings.PUBLIC_BASE_URL:
        return f"{settings.PUBLIC_BASE_URL.rstrip('/')}/storage/{storage_key}"
    return None
```

- [ ] **Step 4: Run — expect pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_media_url.py -v`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git add src/utils.py tests/integration/test_social_media_url.py
git commit -m "feat(social): social media public-URL resolver (LOCAL_MEDIA_BASE_URL, Spaces-safe)"
```

---

## Task 8: `MetaGraphClient` publish methods + richer error

**Files:**
- Modify: `src/integrations/meta_graph.py`

- [ ] **Step 1: Upgrade `MetaGraphError` and add `_post`**

Replace the `MetaGraphError` class in `src/integrations/meta_graph.py` with:

```python
class MetaGraphError(Exception):
    def __init__(
        self,
        message: str,
        *,
        code: Optional[int] = None,
        error_subcode: Optional[int] = None,
        http_status: Optional[int] = None,
        fbtrace_id: Optional[str] = None,
        headers: Optional[dict[str, str]] = None,
        body: Optional[dict[str, Any]] = None,
    ) -> None:
        super().__init__(message)
        self.message = message
        self.code = code
        self.error_subcode = error_subcode
        self.http_status = http_status
        self.fbtrace_id = fbtrace_id
        self.headers = headers or {}
        self.body = body or {}
```

Add a `_raise_graph_error` helper and a `_post` method to `MetaGraphClient` (place after `_get`):

```python
    def _raise_graph_error(self, path: str, resp: httpx.Response) -> None:
        try:
            body = resp.json()
        except ValueError:
            body = {}
        error = body.get("error", {}) if isinstance(body, dict) else {}
        logger.warning(
            "meta_graph.error", path=path, status=resp.status_code, body=resp.text
        )
        raise MetaGraphError(
            error.get("message") or f"Meta Graph {path} -> {resp.status_code}",
            code=error.get("code"),
            error_subcode=error.get("error_subcode"),
            http_status=resp.status_code,
            fbtrace_id=error.get("fbtrace_id"),
            headers=dict(resp.headers),
            body=body if isinstance(body, dict) else {},
        )

    async def _post(self, path: str, data: dict[str, Any]) -> dict[str, Any]:
        async with httpx.AsyncClient(timeout=60) as client:
            resp = await client.post(f"{self.base_url}{path}", data=data)
        if resp.status_code not in (200, 201):
            self._raise_graph_error(path, resp)
        return resp.json()
```

Also update the existing `_get` to call `_raise_graph_error` instead of raising the bare string, so GET errors carry the structured fields:

```python
    async def _get(self, path: str, params: dict[str, Any]) -> dict[str, Any]:
        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(f"{self.base_url}{path}", params=params)
        if resp.status_code != 200:
            self._raise_graph_error(path, resp)
        return resp.json()
```

- [ ] **Step 2: Add Facebook publish methods** (append to `MetaGraphClient`)

```python
    async def publish_facebook_text(
        self, page_id: str, page_token: str, message: str
    ) -> str:
        data = await self._post(
            f"/{page_id}/feed", {"message": message, "access_token": page_token}
        )
        return data["id"]

    async def publish_facebook_single_photo(
        self, page_id: str, page_token: str, image_url: str, message: str
    ) -> str:
        data = await self._post(
            f"/{page_id}/photos",
            {"url": image_url, "caption": message, "access_token": page_token},
        )
        return data.get("post_id") or data["id"]

    async def _upload_unpublished_photo(
        self, page_id: str, page_token: str, image_url: str
    ) -> str:
        data = await self._post(
            f"/{page_id}/photos",
            {"url": image_url, "published": "false", "access_token": page_token},
        )
        return data["id"]

    async def publish_facebook_multi_photo(
        self, page_id: str, page_token: str, image_urls: list[str], message: str
    ) -> str:
        media_ids = [
            await self._upload_unpublished_photo(page_id, page_token, url)
            for url in image_urls
        ]
        attached = {
            f"attached_media[{index}]": json.dumps({"media_fbid": media_id})
            for index, media_id in enumerate(media_ids)
        }
        data = await self._post(
            f"/{page_id}/feed",
            {"message": message, "access_token": page_token, **attached},
        )
        return data["id"]
```

Add `import json` to the top of the file (stdlib group).

- [ ] **Step 3: Add Instagram publish methods** (append to `MetaGraphClient`)

```python
    async def create_instagram_image_container(
        self, ig_id: str, page_token: str, image_url: str, caption: str
    ) -> str:
        data = await self._post(
            f"/{ig_id}/media",
            {"image_url": image_url, "caption": caption, "access_token": page_token},
        )
        return data["id"]

    async def create_instagram_carousel_item(
        self, ig_id: str, page_token: str, image_url: str
    ) -> str:
        data = await self._post(
            f"/{ig_id}/media",
            {
                "image_url": image_url,
                "is_carousel_item": "true",
                "access_token": page_token,
            },
        )
        return data["id"]

    async def create_instagram_carousel(
        self, ig_id: str, page_token: str, child_ids: list[str], caption: str
    ) -> str:
        data = await self._post(
            f"/{ig_id}/media",
            {
                "media_type": "CAROUSEL",
                "children": ",".join(child_ids),
                "caption": caption,
                "access_token": page_token,
            },
        )
        return data["id"]

    async def get_instagram_container_status(
        self, creation_id: str, page_token: str
    ) -> str:
        data = await self._get(
            f"/{creation_id}",
            {"fields": "status_code", "access_token": page_token},
        )
        return data.get("status_code", "")

    async def publish_instagram_container(
        self, ig_id: str, page_token: str, creation_id: str
    ) -> str:
        data = await self._post(
            f"/{ig_id}/media_publish",
            {"creation_id": creation_id, "access_token": page_token},
        )
        return data["id"]

    async def get_instagram_publishing_limit(
        self, ig_id: str, page_token: str
    ) -> dict[str, Any]:
        return await self._get(
            f"/{ig_id}/content_publishing_limit",
            {"fields": "config,quota_usage", "access_token": page_token},
        )
```

- [ ] **Step 4: Verify it imports and the existing monitor tests still pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.integrations.meta_graph import MetaGraphClient, MetaGraphError; e=MetaGraphError('x', code=190); print(e.code, e.headers)" && venv/bin/pytest tests/integration/test_meta_token_monitor.py -q`
Expected: prints `190 {}` and the monitor tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/integrations/meta_graph.py
git commit -m "feat(social): Meta Graph publish methods (FB+IG) and structured MetaGraphError"
```

---

## Task 9: Meta error classification

**Files:**
- Create: `src/services/meta_error_mapping.py`
- Test: `tests/integration/test_meta_error_mapping.py`

- [ ] **Step 1: Write the failing tests**

Create `tests/integration/test_meta_error_mapping.py`:

The expired-token classification is exercised end-to-end by the executor test; here we keep the three that isolate non-DB logic — rate-limit reset parsing, media-invalid subcode classification, and transient retryability.

```python
import json
from datetime import datetime, timezone

from src.constants import PublishErrorCode
from src.integrations.meta_graph import MetaGraphError
from src.services.meta_error_mapping import classify_meta_error

NOW = datetime(2026, 6, 9, 12, 0, tzinfo=timezone.utc)


def test_rate_limit_is_retryable_with_reset_from_header():
    headers = {
        "x-business-use-case-usage": json.dumps(
            {"123": [{"estimated_time_to_regain_access": 30}]}
        )
    }
    outcome = classify_meta_error(MetaGraphError("limit", code=80007, headers=headers), NOW)
    assert outcome.error_code == PublishErrorCode.RATE_LIMITED.value
    assert outcome.retryable is True
    assert outcome.reset_at == datetime(2026, 6, 9, 12, 30, tzinfo=timezone.utc)


def test_media_invalid_by_subcode():
    outcome = classify_meta_error(
        MetaGraphError("bad image", code=100, error_subcode=2207006), NOW
    )
    assert outcome.error_code == PublishErrorCode.MEDIA_INVALID.value
    assert outcome.retryable is False


def test_transient_5xx_is_retryable():
    outcome = classify_meta_error(MetaGraphError("oops", http_status=503), NOW)
    assert outcome.error_code == PublishErrorCode.TRANSIENT.value
    assert outcome.retryable is True
```

- [ ] **Step 2: Run — expect failure**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_error_mapping.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.services.meta_error_mapping'`

- [ ] **Step 3: Implement `src/services/meta_error_mapping.py`**

```python
import json
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional

from src.constants import (
    META_MEDIA_INVALID_ERROR_CODES,
    META_MEDIA_INVALID_ERROR_SUBCODES,
    META_OAUTH_ERROR_CODES,
    META_OAUTH_ERROR_SUBCODES,
    META_RATE_LIMIT_ERROR_CODES,
    META_TRANSIENT_ERROR_CODES,
    PublishErrorCode,
)
from src.integrations.meta_graph import MetaGraphError

RATE_LIMIT_DEFAULT_RESET_SECONDS = 60 * 60
HTTP_SERVER_ERROR_FLOOR = 500
USAGE_HEADERS = ("x-business-use-case-usage", "x-app-usage")


@dataclass
class PublishErrorOutcome:
    error_code: str
    retryable: bool
    reset_at: Optional[datetime]
    message: str


def _is_oauth(error: MetaGraphError) -> bool:
    return (
        error.code in META_OAUTH_ERROR_CODES
        or error.error_subcode in META_OAUTH_ERROR_SUBCODES
    )


def _is_media_invalid(error: MetaGraphError) -> bool:
    return (
        error.code in META_MEDIA_INVALID_ERROR_CODES
        or error.error_subcode in META_MEDIA_INVALID_ERROR_SUBCODES
    )


def _is_transient(error: MetaGraphError) -> bool:
    if error.code in META_TRANSIENT_ERROR_CODES:
        return True
    return bool(error.http_status and error.http_status >= HTTP_SERVER_ERROR_FLOOR)


def _minutes_to_regain(headers: dict[str, str]) -> Optional[int]:
    lowered = {key.lower(): value for key, value in headers.items()}
    for header_name in USAGE_HEADERS:
        raw = lowered.get(header_name)
        if not raw:
            continue
        minutes = _extract_estimated_minutes(raw)
        if minutes is not None:
            return minutes
    return None


def _extract_estimated_minutes(raw: str) -> Optional[int]:
    try:
        parsed = json.loads(raw)
    except ValueError:
        return None
    entries: list = []
    if isinstance(parsed, dict):
        for value in parsed.values():
            entries.extend(value if isinstance(value, list) else [value])
    elif isinstance(parsed, list):
        entries = parsed
    for entry in entries:
        if isinstance(entry, dict) and "estimated_time_to_regain_access" in entry:
            return int(entry["estimated_time_to_regain_access"])
    return None


def _rate_limit_reset_at(error: MetaGraphError, now: datetime) -> datetime:
    minutes = _minutes_to_regain(error.headers)
    if minutes:
        return now + timedelta(minutes=minutes)
    return now + timedelta(seconds=RATE_LIMIT_DEFAULT_RESET_SECONDS)


def classify_meta_error(
    error: MetaGraphError, now: datetime
) -> PublishErrorOutcome:
    platform_message = error.message or "publishing failed"
    if _is_oauth(error):
        return PublishErrorOutcome(
            PublishErrorCode.EXPIRED_TOKEN.value,
            False,
            None,
            "Your connection has expired. Reconnect at Settings -> Social to keep publishing.",
        )
    if error.code in META_RATE_LIMIT_ERROR_CODES:
        reset_at = _rate_limit_reset_at(error, now)
        return PublishErrorOutcome(
            PublishErrorCode.RATE_LIMITED.value,
            True,
            reset_at,
            f"Publishing limit reached. Next slot ~{reset_at.isoformat()}.",
        )
    if _is_media_invalid(error):
        return PublishErrorOutcome(
            PublishErrorCode.MEDIA_INVALID.value,
            False,
            None,
            f"Media rejected: {platform_message}",
        )
    if _is_transient(error):
        return PublishErrorOutcome(
            PublishErrorCode.TRANSIENT.value, True, None, "Temporary error; retrying."
        )
    return PublishErrorOutcome(
        PublishErrorCode.UNKNOWN.value,
        False,
        None,
        f"Publishing failed: {platform_message}",
    )
```

- [ ] **Step 4: Run — expect pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_meta_error_mapping.py -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add src/services/meta_error_mapping.py tests/integration/test_meta_error_mapping.py
git commit -m "feat(social): classify Meta errors into expired/rate-limit/media/transient/unknown"
```

---

# PHASE 2 — Service + API

## Task 10: Schemas

**Files:**
- Create: `src/schemas/posts.py`

- [ ] **Step 1: Create `src/schemas/posts.py`**

```python
import uuid
from datetime import datetime
from typing import Any, Optional

from pydantic import BaseModel


class SocialPublishValidationError(ValueError):
    def __init__(self, status_code: int, detail: Any) -> None:
        message = detail if isinstance(detail, str) else detail.get("message", "Invalid request")
        super().__init__(message)
        self.status_code = status_code
        self.detail = detail


class PostPublishRequest(BaseModel):
    content: Optional[str] = None
    media_ids: list[uuid.UUID] = []
    account_ids: list[uuid.UUID]


class PostScheduleRequest(PostPublishRequest):
    scheduled_at: datetime


class MediaResponse(BaseModel):
    id: uuid.UUID
    public_url: Optional[str]
    content_type: str
    width: Optional[int]
    height: Optional[int]
    byte_size: int
    position: int

    model_config = {"from_attributes": True}


class PostTargetResponse(BaseModel):
    id: uuid.UUID
    platform: str
    social_account_id: uuid.UUID
    status: str
    provider_post_id: Optional[str]
    attempt_count: int
    next_attempt_at: Optional[datetime]
    rate_limit_reset_at: Optional[datetime]
    last_error_code: Optional[str]
    last_error_detail: Optional[str]
    published_at: Optional[datetime]

    model_config = {"from_attributes": True}


class PostResponse(BaseModel):
    id: uuid.UUID
    agency_id: str
    content: Optional[str]
    status: str
    scheduled_at: Optional[datetime]
    published_at: Optional[datetime]
    needs_attention: bool
    created_at: datetime
    media: list[MediaResponse] = []
    targets: list[PostTargetResponse] = []

    model_config = {"from_attributes": True}
```

- [ ] **Step 2: Verify import**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.schemas.posts import PostPublishRequest, PostScheduleRequest, PostResponse, SocialPublishValidationError; print('ok')"`
Expected: `ok`

- [ ] **Step 3: Commit**

```bash
git add src/schemas/posts.py
git commit -m "feat(social): request/response schemas for posts"
```

---

## Task 11: `SocialPublishService` executor

This is the core. It validates a publish request, builds the post + targets + media, and publishes each target with full error mapping, attempt logging, and retry bookkeeping. Build it in three commits: (11a) media validation + post building, (11b) per-target publish + error handling, (11c) simulation mode.

**Files:**
- Create: `src/services/social_publish_service.py`
- Modify: `tests/fakes/meta_graph.py`, `tests/factories/social.py`
- Test: `tests/integration/test_social_publish_service.py`

- [ ] **Step 1: Add a publishing fake graph client** — append to `tests/fakes/meta_graph.py`:

```python
class FakePublishGraphClient:
    def __init__(self, error: Exception | None = None, post_id: str = "POST123"):
        self._error = error
        self._post_id = post_id
        self.calls: list[str] = []

    async def publish_facebook_text(self, page_id, page_token, message) -> str:
        return self._do("fb_text")

    async def publish_facebook_single_photo(self, page_id, page_token, image_url, message) -> str:
        return self._do("fb_single")

    async def publish_facebook_multi_photo(self, page_id, page_token, image_urls, message) -> str:
        return self._do("fb_multi")

    async def create_instagram_image_container(self, ig_id, page_token, image_url, caption) -> str:
        self.calls.append("ig_container")
        if self._error:
            raise self._error
        return "CONTAINER1"

    async def create_instagram_carousel_item(self, ig_id, page_token, image_url) -> str:
        self.calls.append("ig_child")
        return "CHILD1"

    async def create_instagram_carousel(self, ig_id, page_token, child_ids, caption) -> str:
        self.calls.append("ig_carousel")
        return "CONTAINER1"

    async def get_instagram_container_status(self, creation_id, page_token) -> str:
        return "FINISHED"

    async def publish_instagram_container(self, ig_id, page_token, creation_id) -> str:
        return self._do("ig_publish")

    def _do(self, label: str) -> str:
        self.calls.append(label)
        if self._error:
            raise self._error
        return self._post_id
```

- [ ] **Step 2: Add an account factory** — append to `tests/factories/social.py`:

```python
async def seed_connected_account(
    session: AsyncSession,
    agency_id: str,
    platform: str,
    provider_id: str,
    now: datetime,
) -> AgencySocialAccount:
    connection = await _get_or_create_connection(session, agency_id, now)
    account = AgencySocialAccount(
        agency_id=agency_id,
        platform=platform,
        provider_id=provider_id,
        access_token=encrypt("page-token"),
        is_primary=platform == MetaPlatform.FACEBOOK.value,
        authorized=True,
    )
    connection.accounts.append(account)
    await session.flush()
    await session.refresh(account)
    return account


async def _get_or_create_connection(
    session: AsyncSession, agency_id: str, now: datetime
) -> AgencySocialConnection:
    from sqlalchemy import select

    existing = await session.scalar(
        select(AgencySocialConnection).where(
            AgencySocialConnection.agency_id == agency_id
        )
    )
    if existing:
        return existing
    connection = AgencySocialConnection(
        agency_id=agency_id,
        user_access_token=encrypt("user-token"),
        token_expires_at=now + timedelta(days=40),
        status=MetaConnectionStatus.CONNECTED.value,
        warning_level=0,
    )
    session.add(connection)
    await session.flush()
    return connection
```

- [ ] **Step 3: Write the failing test (Facebook single-image happy path)**

Create `tests/integration/test_social_publish_service.py`:

```python
import uuid
from datetime import datetime, timezone

import pytest
from sqlalchemy.ext.asyncio import AsyncSession

from src.constants import MetaPlatform, PostStatus, PostTargetStatus, PublishErrorCode
from src.integrations.meta_graph import MetaGraphError
from src.models.media import Media
from src.models.post import Post
from src.models.post_target import PostTarget
from src.services.social_publish_service import SocialPublishService
from tests.factories.social import seed_connected_account
from tests.fakes.meta_graph import FakePublishGraphClient

NOW = datetime(2026, 6, 9, 12, 0, tzinfo=timezone.utc)
AGENCY = "default"


async def _make_post(session, account, with_media=True):
    post = Post(agency_id=AGENCY, content="Hello", status=PostStatus.PUBLISHING.value)
    session.add(post)
    await session.flush()
    if with_media:
        session.add(
            Media(
                post_id=post.id,
                agency_id=AGENCY,
                storage_key=f"posts/{AGENCY}/img.jpg",
                public_url="https://cdn.test/img.jpg",
                content_type="image/jpeg",
                byte_size=1000,
                position=0,
            )
        )
    target = PostTarget(
        post_id=post.id,
        social_account_id=account.id,
        platform=account.platform,
        status=PostTargetStatus.PENDING.value,
    )
    session.add(target)
    await session.flush()
    await session.refresh(post)
    await session.refresh(target)
    return post, target


@pytest.mark.asyncio
async def test_facebook_single_image_publishes(db_session: AsyncSession):
    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.FACEBOOK.value, "PAGE1", NOW)
    post, target = await _make_post(db_session, account)
    graph = FakePublishGraphClient(post_id="FBPOST")

    service = SocialPublishService(db_session, graph_client=graph)
    await service.publish_post(post, NOW)

    await db_session.refresh(target)
    assert target.status == PostTargetStatus.PUBLISHED.value
    assert target.provider_post_id == "FBPOST"
    assert "fb_single" in graph.calls
    await db_session.refresh(post)
    assert post.status == PostStatus.PUBLISHED.value
```

- [ ] **Step 4: Run — expect failure**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_publish_service.py::test_facebook_single_image_publishes -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.services.social_publish_service'`

- [ ] **Step 5: Implement `src/services/social_publish_service.py`**

```python
import asyncio
from datetime import datetime, timedelta
from typing import Optional

import structlog
from sqlalchemy.ext.asyncio import AsyncSession

from src.config import settings
from src.constants import (
    MetaConnectionStatus,
    MetaPlatform,
    PostStatus,
    PostTargetStatus,
    PublishErrorCode,
)
from src.db.repositories.agency_social_account import AgencySocialAccountRepository
from src.db.repositories.agency_social_connection import (
    AgencySocialConnectionRepository,
)
from src.db.repositories.media import MediaRepository
from src.db.repositories.post import PostRepository
from src.db.repositories.post_publish_attempt import PostPublishAttemptRepository
from src.db.repositories.post_target import PostTargetRepository
from src.integrations.meta_graph import MetaGraphClient, MetaGraphError
from src.models.media import Media
from src.models.post import Post
from src.models.post_publish_attempt import PostPublishAttempt
from src.models.post_target import PostTarget
from src.services.meta_error_mapping import classify_meta_error
from src.services.runtime_settings import is_messaging_simulation_enabled
from src.utils import decrypt, resolve_social_public_media_url

logger = structlog.get_logger()

IG_STATUS_FINISHED = "FINISHED"
IG_STATUS_ERROR = "ERROR"


class SocialPublishService:
    def __init__(
        self,
        session: AsyncSession,
        graph_client: Optional[MetaGraphClient] = None,
        clock: Optional[datetime] = None,
    ) -> None:
        self.session = session
        self.graph = graph_client or MetaGraphClient()
        self.posts = PostRepository(session)
        self.targets = PostTargetRepository(session)
        self.media = MediaRepository(session)
        self.attempts = PostPublishAttemptRepository(session)
        self.accounts = AgencySocialAccountRepository(session)
        self.connections = AgencySocialConnectionRepository(session)

    async def publish_post(self, post: Post, now: datetime) -> Post:
        simulated = await is_messaging_simulation_enabled(self.session)
        media_urls = self._media_urls(post)
        for target in post.targets:
            if target.status in (
                PostTargetStatus.PUBLISHED.value,
                PostTargetStatus.NEEDS_RECONNECT.value,
            ):
                continue
            await self.publish_target(target, post, media_urls, now, simulated)
        await self._recompute_post_status(post, now)
        return post

    async def publish_target(
        self,
        target: PostTarget,
        post: Post,
        media_urls: list[str],
        now: datetime,
        simulated: bool,
    ) -> None:
        target.status = PostTargetStatus.PUBLISHING.value
        target.attempt_count += 1
        await self.session.flush()
        account = await self.accounts.get_by_id(target.social_account_id)
        if account is None:
            await self._fail(target, post, PublishErrorCode.UNKNOWN.value, "Account missing", now, None)
            return
        if simulated:
            await self._succeed(target, post, f"sim_{target.id.hex[:10]}", now, simulated=True)
            return
        await self._attempt_real_publish(target, post, account, media_urls, now)

    async def _attempt_real_publish(self, target, post, account, media_urls, now):
        started = now
        try:
            provider_post_id = await self._dispatch(target, account, post.content or "", media_urls)
        except MetaGraphError as error:
            await self._handle_error(target, post, account, error, now, started)
            return
        await self._succeed(target, post, provider_post_id, now, simulated=False, started=started)

    async def _dispatch(self, target, account, content, media_urls) -> str:
        token = decrypt(account.access_token)
        if account.platform == MetaPlatform.FACEBOOK.value:
            return await self._dispatch_facebook(account, token, content, media_urls)
        return await self._dispatch_instagram(target, account, token, content, media_urls)

    async def _dispatch_facebook(self, account, token, content, media_urls) -> str:
        if not media_urls:
            return await self.graph.publish_facebook_text(account.provider_id, token, content)
        if len(media_urls) == 1:
            return await self.graph.publish_facebook_single_photo(
                account.provider_id, token, media_urls[0], content
            )
        return await self.graph.publish_facebook_multi_photo(
            account.provider_id, token, media_urls, content
        )

    async def _dispatch_instagram(self, target, account, token, content, media_urls) -> str:
        creation_id = target.provider_container_id or await self._build_ig_container(
            account, token, content, media_urls
        )
        target.provider_container_id = creation_id
        await self.session.flush()
        return await self.graph.publish_instagram_container(account.provider_id, token, creation_id)

    async def _build_ig_container(self, account, token, content, media_urls) -> str:
        if len(media_urls) == 1:
            creation_id = await self.graph.create_instagram_image_container(
                account.provider_id, token, media_urls[0], content
            )
        else:
            children = [
                await self.graph.create_instagram_carousel_item(account.provider_id, token, url)
                for url in media_urls
            ]
            creation_id = await self.graph.create_instagram_carousel(
                account.provider_id, token, children, content
            )
        await self._await_ig_ready(account, token, creation_id)
        return creation_id

    async def _await_ig_ready(self, account, token, creation_id) -> None:
        for _ in range(settings.SOCIAL_IG_STATUS_POLL_ATTEMPTS):
            status = await self.graph.get_instagram_container_status(creation_id, token)
            if status == IG_STATUS_FINISHED:
                return
            if status == IG_STATUS_ERROR:
                raise MetaGraphError("Instagram container processing failed", code=100)
            await asyncio.sleep(settings.SOCIAL_IG_STATUS_POLL_INTERVAL_SECONDS)

    def _media_urls(self, post: Post) -> list[str]:
        urls: list[str] = []
        for media in sorted(post.media, key=lambda item: item.position):
            resolved = resolve_social_public_media_url(media.storage_key, media.public_url)
            if resolved:
                urls.append(resolved)
        return urls

    async def _succeed(self, target, post, provider_post_id, now, *, simulated, started=None):
        target.status = PostTargetStatus.PUBLISHED.value
        target.provider_post_id = provider_post_id
        target.published_at = now
        target.last_error_code = None
        target.last_error_detail = None
        target.next_attempt_at = None
        await self._log_attempt(target, "success", now, provider_post_id=provider_post_id, simulated=simulated, started=started)
        logger.info("social_publish.success", target_id=str(target.id), platform=target.platform)

    async def _handle_error(self, target, post, account, error, now, started):
        outcome = classify_meta_error(error, now)
        if outcome.error_code == PublishErrorCode.EXPIRED_TOKEN.value:
            account.authorized = False
            await self.connections.update_by_agency(
                account.agency_id, status=MetaConnectionStatus.NEEDS_RECONNECT.value
            )
        target.status = self._status_for(outcome.error_code)
        target.last_error_code = outcome.error_code
        target.last_error_detail = outcome.message
        target.rate_limit_reset_at = outcome.reset_at
        target.next_attempt_at = self._next_attempt(target, outcome, now) if outcome.retryable else None
        await self._log_attempt(
            target, "failure", now, error=error, outcome=outcome, started=started
        )
        logger.warning(
            "social_publish.failure",
            target_id=str(target.id),
            error_code=outcome.error_code,
            retryable=outcome.retryable,
        )

    async def _fail(self, target, post, error_code, message, now, started):
        target.status = PostTargetStatus.FAILED.value
        target.last_error_code = error_code
        target.last_error_detail = message
        target.next_attempt_at = None
        await self._log_attempt(target, "failure", now, started=started, error_detail=message, error_code=error_code)

    def _status_for(self, error_code: str) -> str:
        if error_code == PublishErrorCode.EXPIRED_TOKEN.value:
            return PostTargetStatus.NEEDS_RECONNECT.value
        if error_code == PublishErrorCode.RATE_LIMITED.value:
            return PostTargetStatus.RATE_LIMITED.value
        return PostTargetStatus.FAILED.value

    def _next_attempt(self, target, outcome, now) -> datetime:
        backoff = settings.SOCIAL_POST_RETRY_BASE_SECONDS * (2 ** (target.attempt_count - 1))
        capped = min(backoff, settings.SOCIAL_POST_RETRY_MAX_BACKOFF_SECONDS)
        base = now + timedelta(seconds=capped)
        if outcome.reset_at and outcome.reset_at > base:
            return outcome.reset_at
        return base

    async def _log_attempt(
        self,
        target,
        outcome_label,
        now,
        *,
        provider_post_id=None,
        error=None,
        outcome=None,
        error_code=None,
        error_detail=None,
        simulated=False,
        started=None,
    ):
        account = await self.accounts.get_by_id(target.social_account_id)
        duration = int((now - started).total_seconds() * 1000) if started else None
        attempt = PostPublishAttempt(
            post_id=target.post_id,
            post_target_id=target.id,
            platform=target.platform,
            account_provider_id=account.provider_id if account else "",
            outcome=outcome_label,
            provider_post_id=provider_post_id,
            error_code=outcome.error_code if outcome else error_code,
            error_detail=outcome.message if outcome else error_detail,
            http_status=error.http_status if error else None,
            raw_response=(error.body if error else None),
            duration_ms=duration,
            simulated=simulated,
        )
        self.session.add(attempt)
        await self.session.flush()

    async def _recompute_post_status(self, post: Post, now: datetime) -> None:
        statuses = [target.status for target in post.targets]
        published = all(status == PostTargetStatus.PUBLISHED.value for status in statuses)
        any_published = any(status == PostTargetStatus.PUBLISHED.value for status in statuses)
        pending = any(
            status in (PostTargetStatus.PENDING.value, PostTargetStatus.PUBLISHING.value, PostTargetStatus.RATE_LIMITED.value)
            for status in statuses
        )
        post.status = self._post_status(published, any_published, pending)
        post.needs_attention = post.status in (PostStatus.FAILED.value, PostStatus.PARTIALLY_FAILED.value)
        if published:
            post.published_at = now
        await self.session.flush()

    def _post_status(self, published, any_published, pending) -> str:
        if published:
            return PostStatus.PUBLISHED.value
        if pending:
            return PostStatus.PUBLISHING.value
        if any_published:
            return PostStatus.PARTIALLY_FAILED.value
        return PostStatus.FAILED.value
```

- [ ] **Step 6: Run the happy-path test — expect pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_publish_service.py::test_facebook_single_image_publishes -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add src/services/social_publish_service.py tests/fakes/meta_graph.py tests/factories/social.py tests/integration/test_social_publish_service.py
git commit -m "feat(social): SocialPublishService executor with FB/IG dispatch and attempt logging"
```

- [ ] **Step 8: Add the error-path + IG + sim tests** (append to `tests/integration/test_social_publish_service.py`)

```python
@pytest.mark.asyncio
async def test_expired_token_flags_needs_reconnect(db_session: AsyncSession):
    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.FACEBOOK.value, "PAGE1", NOW)
    post, target = await _make_post(db_session, account)
    graph = FakePublishGraphClient(error=MetaGraphError("expired", code=190, error_subcode=463))

    await SocialPublishService(db_session, graph_client=graph).publish_post(post, NOW)

    await db_session.refresh(target)
    await db_session.refresh(account)
    assert target.status == PostTargetStatus.NEEDS_RECONNECT.value
    assert target.last_error_code == PublishErrorCode.EXPIRED_TOKEN.value
    assert account.authorized is False
    attempts = await SocialPublishService(db_session, graph_client=graph).attempts.list_for_post(post.id)
    assert attempts and attempts[-1].outcome == "failure"


@pytest.mark.asyncio
async def test_rate_limit_sets_retry(db_session: AsyncSession):
    import json

    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.INSTAGRAM.value, "IG1", NOW)
    post, target = await _make_post(db_session, account)
    headers = {"x-app-usage": json.dumps([{"estimated_time_to_regain_access": 30}])}
    graph = FakePublishGraphClient(error=MetaGraphError("limit", code=80007, headers=headers))

    await SocialPublishService(db_session, graph_client=graph).publish_post(post, NOW)

    await db_session.refresh(target)
    assert target.status == PostTargetStatus.RATE_LIMITED.value
    assert target.rate_limit_reset_at == datetime(2026, 6, 9, 12, 30, tzinfo=timezone.utc)
    assert target.next_attempt_at is not None


@pytest.mark.asyncio
async def test_instagram_single_image_publishes(db_session: AsyncSession):
    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.INSTAGRAM.value, "IG1", NOW)
    post, target = await _make_post(db_session, account)
    graph = FakePublishGraphClient(post_id="IGPOST")

    await SocialPublishService(db_session, graph_client=graph).publish_post(post, NOW)

    await db_session.refresh(target)
    assert target.status == PostTargetStatus.PUBLISHED.value
    assert target.provider_post_id == "IGPOST"
    assert target.provider_container_id == "CONTAINER1"
    assert "ig_container" in graph.calls and "ig_publish" in graph.calls
```

- [ ] **Step 9: Run the full service test file — expect pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_publish_service.py -v`
Expected: PASS (4 tests)

- [ ] **Step 10: Commit**

```bash
git add tests/integration/test_social_publish_service.py
git commit -m "test(social): executor error-mapping, IG, and rate-limit retry behavior"
```

---

## Task 12: Permission

**Files:**
- Modify: `src/auth/permissions.py`

- [ ] **Step 1: Add `posts:publish` to owner, admin, agent**

In `src/auth/permissions.py`, add `"posts:publish",` to each of the `"owner"`, `"admin"`, and `"agent"` permission sets in `ROLE_PERMISSIONS`.

- [ ] **Step 2: Verify**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "from src.auth.permissions import has_permission; print(has_permission('agent','posts:publish'), has_permission('owner','posts:publish'), has_permission('admin','posts:publish'))"`
Expected: `True True True`

- [ ] **Step 3: Commit**

```bash
git add src/auth/permissions.py
git commit -m "feat(social): add posts:publish permission to owner/admin/agent"
```

---

## Task 13: API router

The router exposes the 6 endpoints. Validation lives in the service. Build in two commits: (13a) the service create/validate methods + media upload, (13b) the router + endpoint tests.

**Files:**
- Modify: `src/services/social_publish_service.py` (add request handling), `src/main.py`
- Create: `src/api/posts.py`
- Test: `tests/integration/test_api_posts.py`

- [ ] **Step 1: Add request-handling methods to `SocialPublishService`** (append inside the class)

```python
    async def create_immediate(self, agency_id, content, media_ids, account_ids, user_id, now):
        post = await self._build_post(agency_id, content, media_ids, account_ids, user_id, now, scheduled_at=None)
        post.status = PostStatus.PUBLISHING.value
        await self.session.flush()
        return await self.publish_post(post, now)

    async def create_scheduled(self, agency_id, content, media_ids, account_ids, user_id, scheduled_at, now):
        if scheduled_at <= now:
            from src.schemas.posts import SocialPublishValidationError

            raise SocialPublishValidationError(422, "scheduled_at must be in the future")
        post = await self._build_post(agency_id, content, media_ids, account_ids, user_id, now, scheduled_at=scheduled_at)
        post.status = PostStatus.SCHEDULED.value
        await self.session.flush()
        await self.session.refresh(post)
        return post

    async def _build_post(self, agency_id, content, media_ids, account_ids, user_id, now, scheduled_at):
        accounts = await self._validate_accounts(agency_id, account_ids)
        media_rows = await self._validate_media(agency_id, media_ids, accounts, content)
        post = Post(
            agency_id=agency_id,
            content=content,
            created_by_user_id=user_id,
            scheduled_at=scheduled_at,
            status=PostStatus.SCHEDULED.value,
        )
        self.session.add(post)
        await self.session.flush()
        for position, media in enumerate(media_rows):
            media.post_id = post.id
            media.position = position
        for account in accounts:
            self.session.add(
                PostTarget(
                    post_id=post.id,
                    social_account_id=account.id,
                    platform=account.platform,
                    status=PostTargetStatus.PENDING.value,
                )
            )
        await self.session.flush()
        await self.session.refresh(post)
        return post

    async def _validate_accounts(self, agency_id, account_ids):
        from src.schemas.posts import SocialPublishValidationError

        if not account_ids:
            raise SocialPublishValidationError(422, "Select at least one account to publish to.")
        accounts = await self.accounts.list_by_ids(account_ids)
        found = {account.id for account in accounts}
        missing = [str(item) for item in account_ids if item not in found]
        if missing:
            raise SocialPublishValidationError(404, f"Unknown account(s): {', '.join(missing)}")
        unusable = [account for account in accounts if account.agency_id != agency_id or not account.authorized]
        if unusable:
            raise SocialPublishValidationError(
                400,
                {
                    "error_code": "account_needs_reconnect",
                    "message": "Reconnect at Settings -> Social to publish.",
                    "accounts": [
                        {"id": str(account.id), "platform": account.platform, "name": account.name}
                        for account in unusable
                    ],
                },
            )
        return accounts

    async def _validate_media(self, agency_id, media_ids, accounts, content):
        from src.schemas.posts import SocialPublishValidationError
        from src.utils import (
            SOCIAL_IG_MAX_ASPECT_RATIO,
            SOCIAL_IG_MIN_ASPECT_RATIO,
        )

        media_rows = await self.media.list_unattached_for_agency(media_ids, agency_id)
        if len(media_rows) != len(media_ids):
            raise SocialPublishValidationError(422, "One or more media items are invalid or already attached.")
        if len(media_rows) > settings.SOCIAL_MAX_CAROUSEL_ITEMS:
            raise SocialPublishValidationError(422, f"A carousel supports at most {settings.SOCIAL_MAX_CAROUSEL_ITEMS} images.")
        ordered = sorted(media_rows, key=lambda media: media_ids.index(media.id))
        has_instagram = any(account.platform == MetaPlatform.INSTAGRAM.value for account in accounts)
        if has_instagram:
            self._validate_instagram_media(ordered, content, SOCIAL_IG_MIN_ASPECT_RATIO, SOCIAL_IG_MAX_ASPECT_RATIO)
        return ordered

    def _validate_instagram_media(self, media_rows, content, min_ratio, max_ratio):
        from src.schemas.posts import SocialPublishValidationError
        from src.utils import SOCIAL_ALLOWED_IMAGE_TYPES

        if not media_rows:
            raise SocialPublishValidationError(422, "Instagram requires at least one image.")
        if content and len(content) > IG_CAPTION_MAX:
            raise SocialPublishValidationError(422, f"Instagram captions are limited to {IG_CAPTION_MAX} characters.")
        for index, media in enumerate(media_rows):
            if media.content_type not in SOCIAL_ALLOWED_IMAGE_TYPES:
                raise SocialPublishValidationError(422, {"error_code": "media_invalid", "message": f"Image {index + 1} must be JPEG or PNG for Instagram.", "media_index": index})
            if media.width and media.height:
                ratio = media.width / media.height
                if ratio < min_ratio or ratio > max_ratio:
                    raise SocialPublishValidationError(422, {"error_code": "media_invalid", "message": f"Image {index + 1} aspect ratio {ratio:.2f}:1 is outside Instagram's 4:5 to 1.91:1 range.", "media_index": index})

    async def store_media_upload(self, agency_id, file_bytes, content_type, key, width, height):
        from src.services.storage import get_storage_service

        storage = get_storage_service(settings)
        stored = storage.upload_file(file_bytes, key, content_type=content_type)
        public_url = stored if stored.startswith("http") else None
        media = Media(
            agency_id=agency_id,
            storage_key=key,
            public_url=public_url,
            content_type=content_type,
            width=width,
            height=height,
            byte_size=len(file_bytes),
        )
        self.session.add(media)
        await self.session.flush()
        await self.session.refresh(media)
        return media
```

Add these module-level constants near the top of `social_publish_service.py` (after the `IG_STATUS_*` constants):

```python
IG_CAPTION_MAX = 2200
FB_CAPTION_MAX = 63206
```

- [ ] **Step 2: Create `src/api/posts.py`**

```python
import uuid
from datetime import datetime, timezone
from io import BytesIO

import structlog
from fastapi import APIRouter, Depends, HTTPException, UploadFile
from PIL import Image, UnidentifiedImageError
from sqlalchemy.ext.asyncio import AsyncSession

from src.api.deps import get_tenant_scoped_db, require_permission
from src.api.meta_social import DEFAULT_AGENCY_ID
from src.config import settings
from src.db.repositories.media import MediaRepository
from src.db.repositories.post import PostRepository
from src.models.user import User
from src.schemas.posts import (
    MediaResponse,
    PostPublishRequest,
    PostResponse,
    PostScheduleRequest,
    SocialPublishValidationError,
)
from src.services.social_publish_service import SocialPublishService
from src.utils import SOCIAL_ALLOWED_IMAGE_TYPES

logger = structlog.get_logger()

router = APIRouter(prefix="/api/posts", tags=["posts"])
EXT_BY_TYPE = {"image/jpeg": ".jpg", "image/png": ".png"}


def _now() -> datetime:
    return datetime.now(timezone.utc)


@router.post("/media", response_model=list[MediaResponse], status_code=201)
async def upload_media(
    files: list[UploadFile],
    session: AsyncSession = Depends(get_tenant_scoped_db),
    user: User = Depends(require_permission("posts:publish")),
) -> list[MediaResponse]:
    service = SocialPublishService(session)
    saved = []
    for file in files:
        saved.append(await _store_one(service, file))
    return [MediaResponse.model_validate(media) for media in saved]


async def _store_one(service: SocialPublishService, file: UploadFile):
    if file.content_type not in SOCIAL_ALLOWED_IMAGE_TYPES:
        raise HTTPException(status_code=422, detail=f"Unsupported type {file.content_type}; use JPEG or PNG.")
    contents = await file.read()
    if not contents:
        raise HTTPException(status_code=422, detail="Empty file.")
    if len(contents) > settings.SOCIAL_MAX_IMAGE_BYTES:
        raise HTTPException(status_code=422, detail=f"Image exceeds {settings.SOCIAL_MAX_IMAGE_BYTES} bytes.")
    width, height = _dimensions(contents)
    key = f"posts/{DEFAULT_AGENCY_ID}/{uuid.uuid4().hex}{EXT_BY_TYPE[file.content_type]}"
    return await service.store_media_upload(DEFAULT_AGENCY_ID, contents, file.content_type, key, width, height)


def _dimensions(contents: bytes) -> tuple[int, int]:
    try:
        with Image.open(BytesIO(contents)) as image:
            return image.size
    except UnidentifiedImageError as exc:
        raise HTTPException(status_code=422, detail="Not a valid image.") from exc


@router.post("/publish", response_model=PostResponse)
async def publish_now(
    body: PostPublishRequest,
    session: AsyncSession = Depends(get_tenant_scoped_db),
    user: User = Depends(require_permission("posts:publish")),
) -> PostResponse:
    service = SocialPublishService(session)
    try:
        post = await service.create_immediate(
            DEFAULT_AGENCY_ID, body.content, body.media_ids, body.account_ids, user.id, _now()
        )
    except SocialPublishValidationError as exc:
        raise HTTPException(status_code=exc.status_code, detail=exc.detail)
    except ValueError as exc:
        raise HTTPException(status_code=400, detail=str(exc))
    return PostResponse.model_validate(post)


@router.post("/schedule", response_model=PostResponse, status_code=202)
async def schedule(
    body: PostScheduleRequest,
    session: AsyncSession = Depends(get_tenant_scoped_db),
    user: User = Depends(require_permission("posts:publish")),
) -> PostResponse:
    service = SocialPublishService(session)
    try:
        post = await service.create_scheduled(
            DEFAULT_AGENCY_ID, body.content, body.media_ids, body.account_ids, user.id, body.scheduled_at, _now()
        )
    except SocialPublishValidationError as exc:
        raise HTTPException(status_code=exc.status_code, detail=exc.detail)
    except ValueError as exc:
        raise HTTPException(status_code=400, detail=str(exc))
    return PostResponse.model_validate(post)


@router.get("", response_model=list[PostResponse])
async def list_posts(
    offset: int = 0,
    limit: int = 50,
    session: AsyncSession = Depends(get_tenant_scoped_db),
    user: User = Depends(require_permission("posts:publish")),
) -> list[PostResponse]:
    posts = await PostRepository(session).list_for_agency(DEFAULT_AGENCY_ID, offset, limit)
    return [PostResponse.model_validate(post) for post in posts]


@router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: uuid.UUID,
    session: AsyncSession = Depends(get_tenant_scoped_db),
    user: User = Depends(require_permission("posts:publish")),
) -> PostResponse:
    post = await PostRepository(session).get_for_agency(post_id, DEFAULT_AGENCY_ID)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return PostResponse.model_validate(post)


@router.delete("/{post_id}", status_code=204)
async def cancel_post(
    post_id: uuid.UUID,
    session: AsyncSession = Depends(get_tenant_scoped_db),
    user: User = Depends(require_permission("posts:publish")),
) -> None:
    from src.constants import PostStatus

    repo = PostRepository(session)
    post = await repo.get_for_agency(post_id, DEFAULT_AGENCY_ID)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    if post.status != PostStatus.SCHEDULED.value:
        raise HTTPException(status_code=409, detail="Only scheduled posts can be cancelled.")
    post.status = PostStatus.CANCELLED.value
```

- [ ] **Step 3: Register the router in `src/main.py`**

Add the import near the other router imports:

```python
from src.api.posts import router as posts_router
```

Add inside the `include_router` block:

```python
    application.include_router(posts_router)
```

- [ ] **Step 4: Write endpoint tests**

Create `tests/integration/test_api_posts.py`:

```python
from datetime import datetime, timezone

import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from src.api.deps import get_tenant_scoped_db
from src.constants import MetaPlatform
from src.main import app
from tests.factories.social import seed_connected_account

AGENCY = "default"
NOW = datetime(2026, 6, 9, 12, 0, tzinfo=timezone.utc)


@pytest_asyncio.fixture
async def posts_client(db_session: AsyncSession):
    async def _override():
        yield db_session

    app.dependency_overrides[get_tenant_scoped_db] = _override
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()


@pytest.mark.asyncio
async def test_instagram_requires_media(posts_client, db_session):
    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.INSTAGRAM.value, "IG1", NOW)
    response = await posts_client.post(
        "/api/posts/publish",
        json={"content": "hi", "media_ids": [], "account_ids": [str(account.id)]},
    )
    assert response.status_code == 422
    assert "Instagram requires at least one image" in response.text


@pytest.mark.asyncio
async def test_unauthorized_account_returns_reconnect_400(posts_client, db_session):
    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.FACEBOOK.value, "PAGE1", NOW)
    account.authorized = False
    await db_session.flush()
    response = await posts_client.post(
        "/api/posts/publish",
        json={"content": "hi", "media_ids": [], "account_ids": [str(account.id)]},
    )
    assert response.status_code == 400
    assert response.json()["detail"]["error_code"] == "account_needs_reconnect"
```

- [ ] **Step 5: Run the endpoint tests — expect pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_api_posts.py -v`
Expected: PASS (2 tests). The dev-mode `get_current_user` returns an owner system user, so `require_permission("posts:publish")` passes.

- [ ] **Step 6: Commit**

```bash
git add src/services/social_publish_service.py src/api/posts.py src/main.py tests/integration/test_api_posts.py
git commit -m "feat(social): posts API (publish, schedule, media upload, list, detail, cancel) + validation"
```

---

# PHASE 3 — Scheduling

## Task 14: Poller + lifespan wiring

**Files:**
- Create: `src/services/social_post_poller.py`
- Modify: `src/main.py`
- Test: `tests/integration/test_social_post_poller.py`

- [ ] **Step 1: Write the failing test**

Create `tests/integration/test_social_post_poller.py`:

```python
from datetime import datetime, timedelta, timezone

import pytest
from sqlalchemy.ext.asyncio import AsyncSession

from src.constants import MetaPlatform, PostStatus, PostTargetStatus
from src.models.post import Post
from src.models.post_target import PostTarget
from src.services.social_post_poller import SocialPostRunner
from tests.factories.social import seed_connected_account
from tests.fakes.meta_graph import FakePublishGraphClient

AGENCY = "default"
NOW = datetime(2026, 6, 9, 12, 0, tzinfo=timezone.utc)


@pytest.mark.asyncio
async def test_due_scheduled_post_is_published(db_session: AsyncSession):
    account = await seed_connected_account(db_session, AGENCY, MetaPlatform.FACEBOOK.value, "PAGE1", NOW)
    post = Post(
        agency_id=AGENCY,
        content="Scheduled",
        status=PostStatus.SCHEDULED.value,
        scheduled_at=NOW - timedelta(minutes=1),
    )
    db_session.add(post)
    await db_session.flush()
    db_session.add(PostTarget(post_id=post.id, social_account_id=account.id, platform=account.platform, status=PostTargetStatus.PENDING.value))
    await db_session.flush()

    graph = FakePublishGraphClient(post_id="FBPOST")
    await SocialPostRunner(db_session, graph_client=graph).run_once(NOW)

    await db_session.refresh(post)
    assert post.status == PostStatus.PUBLISHED.value
```

- [ ] **Step 2: Run — expect failure**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_post_poller.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.services.social_post_poller'`

- [ ] **Step 3: Implement `src/services/social_post_poller.py`**

```python
import asyncio
from datetime import datetime, timedelta, timezone
from typing import Optional

import asyncpg
import structlog
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from src.config import list_available_tenants, settings
from src.constants import PostStatus
from src.db.repositories.media import MediaRepository
from src.db.repositories.post import PostRepository
from src.db.repositories.post_target import PostTargetRepository
from src.db.session import async_session_factory, get_tenant_session_factory
from src.integrations.meta_graph import MetaGraphClient
from src.services.social_publish_service import SocialPublishService
from src.services.storage import get_storage_service

logger = structlog.get_logger()


class SocialPostRunner:
    def __init__(self, session: AsyncSession, graph_client: Optional[MetaGraphClient] = None):
        self.session = session
        self.graph = graph_client or MetaGraphClient()
        self.posts = PostRepository(session)
        self.targets = PostTargetRepository(session)
        self.media = MediaRepository(session)
        self.service = SocialPublishService(session, graph_client=self.graph)

    async def run_once(self, now: datetime) -> None:
        await self._publish_due_scheduled(now)
        await self._retry_due_targets(now)
        await self._sweep_orphan_media(now)

    async def _publish_due_scheduled(self, now: datetime) -> None:
        for post in await self.posts.list_due_scheduled(now):
            post.status = PostStatus.PUBLISHING.value
            await self.session.flush()
            await self.service.publish_post(post, now)

    async def _retry_due_targets(self, now: datetime) -> None:
        simulated = await _simulation_enabled(self.session)
        for target in await self.targets.list_retry_due(now, settings.SOCIAL_POST_MAX_ATTEMPTS):
            post = await self.posts.get_by_id(target.post_id)
            if post is None:
                continue
            await self.service.publish_target(target, post, self.service._media_urls(post), now, simulated)
            await self.service._recompute_post_status(post, now)

    async def _sweep_orphan_media(self, now: datetime) -> None:
        cutoff = now - timedelta(hours=settings.SOCIAL_MEDIA_ORPHAN_TTL_HOURS)
        storage = get_storage_service(settings)
        for media in await self.media.list_orphans(cutoff):
            storage.delete_file(media.storage_key)
            await self.media.delete(media)


async def _simulation_enabled(session: AsyncSession) -> bool:
    from src.services.runtime_settings import is_messaging_simulation_enabled

    return await is_messaging_simulation_enabled(session)


async def _run_for_factory(factory: async_sessionmaker[AsyncSession], client: MetaGraphClient, now: datetime) -> None:
    async with factory() as session:
        await SocialPostRunner(session, client).run_once(now)
        await session.commit()


async def _run_default(client: MetaGraphClient, now: datetime) -> None:
    try:
        await _run_for_factory(async_session_factory, client, now)
    except Exception:
        logger.exception("social_post_poller.default_error")


async def _run_tenant(slug: str, client: MetaGraphClient, now: datetime) -> None:
    try:
        await _run_for_factory(get_tenant_session_factory(slug), client, now)
    except asyncpg.exceptions.InvalidCatalogNameError:
        logger.warning("social_post_poller.tenant_db_missing", tenant=slug)
    except Exception:
        logger.exception("social_post_poller.tenant_error", tenant=slug)


async def social_post_poller_loop() -> None:
    interval = settings.SOCIAL_POST_POLL_INTERVAL_SECONDS
    logger.info("social_post_poller.started", interval=interval)
    while True:
        client = MetaGraphClient()
        now = datetime.now(timezone.utc)
        await _run_default(client, now)
        for slug in list_available_tenants():
            await _run_tenant(slug, client, now)
        await asyncio.sleep(interval)
```

- [ ] **Step 4: Run — expect pass**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_post_poller.py -v`
Expected: PASS (1 test)

- [ ] **Step 5: Wire the loop into `src/main.py` lifespan**

Add the import near `meta_token_monitor_loop`:

```python
from src.services.social_post_poller import social_post_poller_loop
```

In `lifespan`, after the `meta_task = asyncio.create_task(...)` line, add:

```python
    social_task = asyncio.create_task(social_post_poller_loop())
```

And in the shutdown section (after `meta_task.cancel()`), add:

```python
    social_task.cancel()
```

- [ ] **Step 6: Verify the app imports cleanly**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/python -c "import src.main; print('app ok')"`
Expected: `app ok`

- [ ] **Step 7: Commit**

```bash
git add src/services/social_post_poller.py src/main.py tests/integration/test_social_post_poller.py
git commit -m "feat(social): scheduled-post poller with retry + orphan sweep, wired into lifespan"
```

---

## Task 15: Full suite + deploy-rollout checklist

**Files:** none (verification + ops)

- [ ] **Step 1: Run the whole new test surface**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest tests/integration/test_social_media_url.py tests/integration/test_meta_error_mapping.py tests/integration/test_social_publish_service.py tests/integration/test_social_post_poller.py tests/integration/test_api_posts.py tests/integration/test_meta_token_monitor.py -v`
Expected: all PASS, no errors.

- [ ] **Step 2: Run the full test suite to confirm no regressions**

Run: `cd "/Users/binhquach/Workplace/Slicify AI/slicify-realestate" && venv/bin/pytest -q`
Expected: the suite passes (or only pre-existing unrelated failures — compare against `git stash` baseline if unsure).

- [ ] **Step 3: Document the deploy rollout in the PR description**

The PR must instruct the deployer to run, on UAT and prod:

```
SLICIFY_CONTROL_DATABASE_URL=$CONTROL_DB TEMPLATE_DB_URL=$TEMPLATE_DB \
  venv/bin/python scripts/migrate_all_tenants.py
```

and to set `LOCAL_MEDIA_BASE_URL` only in local dev (leave empty in UAT/prod, where DO Spaces returns absolute URLs). No new secrets, so `patch_tenant_envs.py` is optional (run it only if you want `LOCAL_MEDIA_BASE_URL` present in tenant `.env` files for completeness).

- [ ] **Step 4: Final commit (if any docs/PR notes were added)**

```bash
git add -A
git commit -m "chore(social): rollout notes for multi-tenant migration" || true
```

---

## Post-implementation review

After all tasks: run `superpowers:requesting-code-review`. Confirm against the spec's four required behaviors:
1. **Expired token** → `PostTarget.status = needs_reconnect`, `account.authorized=False`, connection → `needs_reconnect`, actionable message (Task 11, `test_expired_token_flags_needs_reconnect`).
2. **Rate limit** → `rate_limited` with `rate_limit_reset_at` + retry scheduled (Task 11, `test_rate_limit_sets_retry`).
3. **Media rejection** → specific 422 at validation, `media_invalid` at publish (Task 13, `test_instagram_requires_media`; Task 9 media mapping).
4. **All attempts logged** → every `publish_target` branch writes a `PostPublishAttempt` (Task 11; asserted in `test_expired_token_flags_needs_reconnect`).
