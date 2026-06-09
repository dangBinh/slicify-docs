# Social Publishing (Facebook / Instagram) — Design

- **Date:** 2026-06-09
- **Branch:** `feature/social-publishing-facebook-instagram` (Linear ticket TBD)
- **App:** Slate (`slicify-realestate`)
- **Status:** Approved design — ready for implementation plan
- **Builds on:** DEV-170 (Meta OAuth + token service) — connections/accounts/tokens already exist

---

## 1. Goal

Let an agency publish a post to its connected **Facebook Page(s)** and **Instagram
Business account(s)** from one place — either **immediately** or **scheduled** for a
future time — reusing the Meta connection already established by DEV-170.

Two named endpoints:

- **`POST /api/posts/publish`** — create the post and fan it out to the chosen accounts now.
- **`POST /api/posts/schedule`** — store the post for later; a background poller fires it at its scheduled time.

The feature must surface the failure modes that silently break social publishing:

1. **Expired token** → actionable error prompting reconnect; never a silent failure.
2. **Rate limit** (Instagram content-publishing limit, 25 posts / 24h) → clear error with time until reset.
3. **Media format/size rejection** → specific validation message.
4. **Every publish attempt** (success and failure) logged with timestamp, platform, and error detail.

## 2. Scope

### In scope (v1)

- Fan-out model: one `Post` → many `PostTarget`s (one per FB Page / IG account), each with its own status + result (partial failures handled per target).
- Image publishing only: single image, multi-image **carousel** (2–10), and **Facebook text-only** posts.
- Media **uploaded into Slate storage** (via the existing storage service), attached to the post.
- Immediate publish (inline) and scheduled publish (asyncio poller, iterating every tenant DB).
- Auto-retry with backoff on retryable failures (rate limit waits until reset).
- Dedicated append-only **publish-attempt log**.
- Meta-error → actionable-error mapping for the three required scenarios.
- A local-dev media-tunnel config flag (`LOCAL_MEDIA_BASE_URL`) so Instagram can fetch locally-stored images via ngrok.

### Deferred (explicitly parked, not missing)

- **Video / Reels** publishing (async container processing, resumable upload, duration/codec validation).
- **Per-platform content variants** (Mixpost "versions" — different caption/media per account).
- **Revision history / edit + rollback** of scheduled posts (and the `PATCH`/restore endpoints it implies).
- **Frontend** composer / Posts UI — built separately; the shared mockup is the behavioral contract.
- **Manual retry** endpoint (auto-retry only in v1).
- **Publishing as an agent skill** (`publish_social_post`) — the executor is structured so a thin skill can wrap it later with no refactor.
- Post analytics/insights; other networks (LinkedIn / X); pre-publish approval workflow.

## 3. Context & constraints (from the codebase)

- **OAuth is done (DEV-170).** Tokens live in two tables scoped by `agency_id`:
  `agency_social_connections` (long-lived user token + expiry/health: `connected | expiring
  | expired | needs_reconnect | error`) and `agency_social_accounts` (one row per connected
  FB Page / IG account, each with its own page token + an `authorized` flag). A background
  `meta_token_monitor_loop` already refreshes tokens and flips connection status.
- **`MetaGraphClient` (`src/integrations/meta_graph.py`) only does OAuth today** — token
  exchange, `get_pages`, `get_ig_account`. **No publishing methods exist.** It has a `_get`
  helper only; `MetaGraphError` is a bare string exception.
- **No `Post` model exists** anywhere — clean slate.
- **No scheduler.** Recurring work is an `asyncio` loop started in FastAPI `lifespan`
  (`watchdog_loop`, `meta_token_monitor_loop`), each iterating every tenant DB. The poller
  follows that exact pattern.
- **Storage service exists** (`src/services/storage.py`, `get_storage_service(settings)`):
  `SpacesStorageService` (DigitalOcean Spaces, prod — `upload_file` returns a full `https://`
  URL but **hardcodes `ContentType="application/pdf"`**) or `LocalStorageService` (dev — writes
  to `<repo>/storage/`, returns a relative key). `/storage` is mounted as public `StaticFiles`
  in `main.py`.
- **Absolute public URLs for local media** are only built in one place today —
  `whatsapp_orchestrator._resolve_public_media_url()` prepends `PUBLIC_BASE_URL` to a
  `/storage/...` key and returns `None` if unset. Instagram needs the same: it can only publish
  from a publicly reachable URL.
- **Image validation helpers exist** — `save_uploaded_image()` in `src/utils.py`
  (content-type allowlist + PIL dimension checks); a 5 MB cap pattern on contact headshots.
  No video validation anywhere.
- **Layered backend rules** (`src/CLAUDE.md`): router → service → repository; Pydantic schemas
  in `src/schemas/`; no comments; no magic literals (named constants); functions < 20 lines;
  `structlog` only; authenticated endpoints use `get_tenant_scoped_db`.

## 4. Key decisions

| Decision | Choice | Rationale |
| --- | --- | --- |
| Post ↔ targets | **Fan-out**: 1 `Post` → many `PostTarget` | Matches Mixpost + "blast a listing everywhere"; per-target status enables partial failure |
| Media source | **Upload into Slate storage** (existing storage service) | Satisfies IG's public-URL requirement via `/storage` mount; reuses existing infra |
| Media types (v1) | **Images only** — single, carousel (2–10), FB text-only | Ships fastest; video/Reels deferred |
| Scheduling | **asyncio poller loop** in `lifespan`, iterating tenants | Matches `meta_token_monitor_loop`; no new infra |
| Scheduled failure | **Auto-retry with backoff** (rate limit waits until reset) | Resilient scheduled delivery |
| Publish log | **Dedicated append-only table** `post_publish_attempts` | Purpose-built for timestamp/platform/error history across retries |
| Execution architecture | **One pipeline, two entry points, one shared executor** | Immediate + scheduled can't drift; single home for platform logic, logging, error mapping |
| Skill exposure (v1) | **Endpoint-only**; executor structured for a later skill wrapper | Smallest surface now |
| Frontend | **Backend-only**; mockup = contract | Composer/Posts UI built separately |
| REST path | **Unversioned** `/api/posts` | Matches every other resource router; only webhooks are versioned in Slate |
| Content versioning | **Deferred** (revision history + per-platform variants) | Out of v1 scope |
| Local-dev media URL | **`LOCAL_MEDIA_BASE_URL`** override (ngrok), exclusive of `PUBLIC_BASE_URL`; Spaces untouched | Lets IG fetch local images in dev without Spaces |

## 5. Data model — four new tables

All scoped by `agency_id` (string, no FK — same convention as `agency_social_connections`).
One model per file under `src/models/`. No secrets stored here (tokens already live on
`agency_social_accounts`).

### 5.1 `Post` — `posts` (the shared content envelope)

| field | type | notes |
| --- | --- | --- |
| `id` | UUID PK | |
| `agency_id` | str(100), indexed | scope key |
| `listing_id` | UUID FK→listings, nullable | optional real-estate context |
| `content` | Text, nullable | shared post text/caption |
| `status` | str(20) | `scheduled` → `publishing` → `published` / `partially_failed` / `failed` / `cancelled` |
| `scheduled_at` | datetime tz, nullable | set only by `/schedule` |
| `published_at` | datetime tz, nullable | when all targets terminal |
| `needs_attention` | bool, default false | flips on terminal failure (mirrors groups pattern) |
| `created_by_user_id` | UUID FK→users, nullable | |
| `created_at` / `updated_at` | timestamps | |

### 5.2 `PostTarget` — `post_targets` (fan-out + per-account retry state)

| field                       | type                           | notes                                                                                        |
| --------------------------- | ------------------------------ | -------------------------------------------------------------------------------------------- |
| `id`                        | UUID PK                        |                                                                                              |
| `post_id`                   | UUID FK→posts CASCADE, indexed |                                                                                              |
| `social_account_id`         | UUID FK→agency_social_accounts | which connected account                                                                      |
| `platform`                  | str(20)                        | `facebook` \| `instagram` (denormalized)                                                     |
| `status`                    | str(20)                        | `pending`→`publishing`→`published` / `failed` / `rate_limited` / `needs_reconnect`           |
| `provider_post_id`          | str(100), nullable             | FB/IG post id on success                                                                     |
| `provider_container_id`     | str(100), nullable             | IG `creation_id` — a retry **resumes publish** instead of re-creating the container          |
| `attempt_count`             | smallint, default 0            |                                                                                              |
| `next_attempt_at`           | datetime tz, nullable          | backoff / rate-limit-until-reset                                                             |
| `rate_limit_reset_at`       | datetime tz, nullable          | from Meta usage headers / `content_publishing_limit`                                         |
| `last_error_code`           | str(50), nullable              | normalized: `expired_token` \| `rate_limited` \| `media_invalid` \| `transient` \| `unknown` |
| `last_error_detail`         | Text, nullable                 | actionable message                                                                           |
| `published_at`              | datetime tz, nullable          |                                                                                              |
| `created_at` / `updated_at` | timestamps                     |                                                                                              |

### 5.3 `Media` — `media` (uploaded images, shared across targets)

| field | type | notes |
| --- | --- | --- |
| `id` | UUID PK | |
| `post_id` | UUID FK→posts CASCADE, **nullable** | NULL while uploaded-but-unattached; set on publish/schedule |
| `agency_id` | str(100), indexed | scope key (set at upload) |
| `storage_key` | str(500) | key returned by the storage service |
| `public_url` | str(1000), nullable | absolute URL (Spaces returns it at upload; local built at publish) |
| `media_type` | str(20) | `image` (v1; `video` reserved) |
| `content_type` | str(100) | `image/jpeg` \| `image/png` |
| `width` / `height` | int, nullable | for IG aspect-ratio validation |
| `byte_size` | int | |
| `position` | smallint, default 0 | carousel ordering |
| `created_at` / `updated_at` | timestamps | |

- 1 media = single image · 2–10 = carousel · 0 = Facebook text-only (IG targets rejected at validation).
- Orphaned rows (`post_id IS NULL` older than `ORPHAN_TTL`) are swept by the poller (row + storage file deleted).

### 5.4 `PostPublishAttempt` — `post_publish_attempts` (append-only audit log)

| field | type | notes |
| --- | --- | --- |
| `id` | UUID PK | |
| `post_id` | UUID FK→posts CASCADE, indexed | |
| `post_target_id` | UUID FK→post_targets CASCADE, indexed | |
| `platform` | str(20) | **required: platform** |
| `account_provider_id` | str(100) | FB page / IG account id at attempt time |
| `attempted_at` | datetime tz, default `clock_timestamp()` | **required: timestamp** |
| `outcome` | str(20) | `success` \| `failure` |
| `provider_post_id` | str(100), nullable | on success |
| `error_code` | str(50), nullable | normalized code |
| `error_detail` | Text, nullable | **required: actionable error detail** |
| `http_status` | smallint, nullable | Meta HTTP status |
| `raw_response` | JSON, nullable | truncated Meta error body for debugging |
| `duration_ms` | int, nullable | |
| `simulated` | bool, default false | true when simulation mode skipped the real send |

One row **per attempt** — retries accumulate full history. `PostTarget` holds *current* state;
`PostPublishAttempt` holds *what happened each try*.

### 5.5 Enums / constants (`src/constants.py`)

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
```

Reuse existing `MetaPlatform` (`facebook` | `instagram`). Meta numeric error codes/subcodes
are kept as named constants (e.g. `META_ERROR_OAUTH = 190`, `IG_ERROR_CONTENT_PUBLISH_LIMIT = 80007`).

## 6. API endpoints

All under `prefix="/api/posts"`, `Depends(get_tenant_scoped_db)`, gated by a new
**`posts:publish`** permission (owner / admin / agent). `agency_id` resolved server-side
(same resolver `meta_social.py` uses). Schemas in `src/schemas/posts.py`. Routers contain
zero business logic — they call `SocialPublishService` / repositories and return a schema.

| Method + path | Purpose | Returns |
| --- | --- | --- |
| **`POST /api/posts/publish`** | named — immediate fan-out | `200` Post + per-target results |
| **`POST /api/posts/schedule`** | named — store for later | `202` Post (`status=scheduled`) |
| `POST /api/posts/media` | upload 1+ images (multipart) → validate + store | `201` list of media refs |
| `GET /api/posts` | Posts tab list (paginated) | Post summaries |
| `GET /api/posts/{id}` | detail: post + targets + attempt history | Post detail |
| `DELETE /api/posts/{id}` | cancel a `scheduled` post before it fires | `204` |

The composer's connected-account toggles reuse the **existing** DEV-170
`GET /api/social/meta/...` connection endpoint — no new accounts endpoint.

### 6.1 Request bodies (JSON)

```
PostPublishRequest:
  content:     str | None
  media_ids:   list[UUID]   # 0–10, must be unattached + owned by agency
  account_ids: list[UUID]   # ≥1 AgencySocialAccount ids (platform toggle → account map done client-side)
  listing_id:  UUID | None

PostScheduleRequest(PostPublishRequest):
  scheduled_at: datetime    # tz-aware, strictly future (≥ 1 min lead)
```

The mockup's `Facebook` / `Instagram` toggles map to `account_ids` client-side using the
connected-accounts list it already loads to render the toggles.

### 6.2 Up-front validation (fail fast, **before any Meta call**)

This is where two of the three required error scenarios are enforced *before* a send:

1. **Accounts** — each `account_id` exists, belongs to the agency, `authorized=True`, and its
   connection isn't `expired` / `needs_reconnect`. Else → **`400`**
   `{error_code:"account_needs_reconnect", message:"Reconnect Facebook at Settings → Social to publish.", accounts:[{id,platform,name}]}`.
   ⇐ *expired-token requirement, caught before we even try.*
2. **Instagram needs media** — any IG account selected with `media_ids == []` →
   `422 "Instagram requires at least one image."`
3. **Caption limits (per selected platform)** — IG selected & `len(content) > 2200` → `422`;
   Facebook cap `63 206`.
4. **Image constraints for IG targets** — content-type ∈ {jpeg, png}, `byte_size ≤ SOCIAL_MAX_IMAGE_BYTES`,
   aspect ratio ∈ [4:5 … 1.91:1] (from stored `width/height`) → else **`422`**
   `{error_code:"media_invalid", message:"Image 2 aspect ratio 2.3:1 exceeds Instagram's 1.91:1 maximum.", media_index:2}`.
   ⇐ *media-rejection requirement, specific message.*
5. **Carousel** — `media_ids` length ≤ `SOCIAL_MAX_CAROUSEL_ITEMS` (10).
6. **Schedule** — `scheduled_at` strictly future → else `422`.

### 6.3 `POST /api/posts/media` (upload)

Multipart files. Per file: content-type allowlist, `≤ SOCIAL_MAX_IMAGE_BYTES`, open with PIL
for `width/height` (catch → `422 "Not a valid image."`), store via the storage service
(content-type **parameterized** — see §10), create a `Media` row with `post_id=NULL`. Returns
`{id, public_url, content_type, width, height, byte_size}` for the live preview. Reuses/extends
the `save_uploaded_image()` validation helpers.

### 6.4 Response shape

`PostResponse`: post fields + `media: [...]` + `targets: [{id, platform, account_name, status,
provider_post_id, attempt_count, next_attempt_at, rate_limit_reset_at, last_error_code,
last_error_detail, published_at}]`.

Immediate `/publish` returns **`200` even on partial failure** — the per-target detail carries
the actionable error (rate-limit reset time, reconnect prompt), and `attempt_count` /
`next_attempt_at` show the poller will auto-retry retryable targets. A whole-request failure
(invalid input, all accounts need reconnect) is a top-level `400`/`422` instead.

## 7. Publishing executor — `SocialPublishService` (`src/services/social_publish_service.py`)

Injects `AsyncSession` + `MetaGraphClient` + repositories. The **one** place platform logic,
logging, error mapping, and retry bookkeeping live — shared by both endpoints and the poller.

`publish_target(target)`:

1. **Claim** — set `status=publishing`, flush (guards double-send; `SELECT … FOR UPDATE SKIP LOCKED` on the poller claim).
2. Resolve account + **decrypt the page token** (reuse the existing meta token-read helper).
3. Resolve each media → **absolute public URL** via `resolve_social_public_media_url()` (§10). IG with no resolvable URL → fail with config-actionable error.
4. **Branch by platform:**
   - **Facebook** — text-only / single photo / multi-photo (see §8).
   - **Instagram** — if `provider_container_id` already set → **resume at publish**; else create container(s) (single or carousel) → poll status until `FINISHED` → store `provider_container_id` → `media_publish`.
5. **Success** → set `provider_post_id`, `status=published`, `published_at`; write `PostPublishAttempt(success)`.
6. **`MetaGraphError`** → `classify_meta_error()` (§9) → update target (`status`, `last_error_code/detail`, `attempt_count++`, `next_attempt_at`, `rate_limit_reset_at`) + write `PostPublishAttempt(failure, http_status, raw_response)`. On `expired_token` also set `account.authorized=False` + connection → `needs_reconnect` (reuse the meta status update path → lights the existing banner).
7. Always stamp `duration_ms` and emit a `structlog` line.

`publish_post(post)` runs targets sequentially (rate-limit-aware; `asyncio.gather` is a later
optimization), then recomputes `Post.status` (`published` if all published; `partially_failed`
on a mix; `failed` if all terminal-failed; stays `publishing` while a retry is pending) and sets
`needs_attention` on any terminal failure.

**Simulation mode** (reuse the messaging sim toggle): skip the Graph call, `provider_post_id="sim_…"`,
`status=published`, `PostPublishAttempt(simulated=true)`.

## 8. `MetaGraphClient` additions (`src/integrations/meta_graph.py`)

Add a `_post()` helper and upgrade `MetaGraphError` to carry `code`, `error_subcode`,
`http_status`, `fbtrace_id`, and response `headers` (parsed from Meta's `{"error":{…}}`
envelope + `X-App-Usage` / `X-Business-Use-Case-Usage`).

| Method | Graph call |
| --- | --- |
| `publish_facebook_text(page_id, tok, msg)` | `POST /{page_id}/feed` |
| `publish_facebook_single_photo(page_id, tok, url, msg)` | `POST /{page_id}/photos` |
| `publish_facebook_multi_photo(page_id, tok, urls, msg)` | N× `/photos {published:false}` → `/feed {attached_media:[…]}` |
| `create_instagram_image_container(ig_id, tok, url, content)` | `POST /{ig_id}/media` |
| `create_instagram_carousel(ig_id, tok, child_ids, content)` | child `/media {is_carousel_item:true}` → parent `/media {media_type:CAROUSEL, children:[…]}` |
| `get_instagram_container_status(creation_id, tok)` | `GET /{creation_id}?fields=status_code` (poll → `FINISHED`) |
| `publish_instagram_container(ig_id, tok, creation_id)` | `POST /{ig_id}/media_publish` |
| `get_instagram_publishing_limit(ig_id, tok)` | `GET /{ig_id}/content_publishing_limit` (25/24h quota → reset computation) |

All calls go through the single `base_url` built from `META_GRAPH_VERSION`, so a Graph-version
bump is one config change (see §13).

## 9. Error mapping — `classify_meta_error()`

Returns `(error_code, retryable, reset_at, actionable_message)`. Meta codes kept as named constants.

| Meta signal | `error_code` | retryable | Target status | Actionable message |
| --- | --- | --- | --- | --- |
| code **190** (+sub 463/467/460/458), `OAuthException` | `expired_token` | **no** | `needs_reconnect` (also flips `account.authorized=False` + connection → `needs_reconnect`) | "Your {platform} connection has expired. Reconnect at Settings → Social to keep publishing." |
| codes **4 / 17 / 32 / 613 / 80007** (IG content-publish limit) | `rate_limited` | **yes** | `rate_limited` | "Instagram limit reached (25 posts/24h). Next slot ~{reset, local}." — `reset_at` from `estimated_time_to_regain_access` header, else `content_publishing_limit`, else +1h |
| code **100** + media subcodes (2207003/04/06, 36xxx), 9004 | `media_invalid` | **no** | `failed` | Meta's specific reason → "Instagram rejected the image: aspect ratio out of range." |
| codes **1 / 2**, HTTP 5xx | `transient` | yes | `failed` → retry | "Temporary {platform} error; retrying." |
| anything else | `unknown` | **no** | `failed` + `needs_attention` | "Publishing failed: {meta message}." |

Every branch — success and all five failure rows — writes a `PostPublishAttempt` row (timestamp,
platform, error_code, error_detail) **and** a `structlog` line. The audit requirement is met at
the storage layer, not just logs.

> **Graph-version sunset hint:** a wholesale `META_GRAPH_VERSION` deprecation surfaces as
> `unknown`/`transient` across *all* targets simultaneously — distinct from a per-account token
> issue. Note this in runbooks so it isn't mistaken for a reconnect problem.

## 10. Media: upload, validation, public-URL resolution

### 10.1 Storage write (DO Spaces untouched)

`SpacesStorageService.upload_file(content, key, content_type="application/pdf")` — content-type
becomes an **optional** param defaulting to the current value, so every existing Spaces caller
(document/PDF generation, Annature) behaves byte-identically. Only the new image upload passes
`content_type="image/jpeg"` / `"image/png"`.

### 10.2 Public-URL resolution (social-only resolver)

A dedicated `resolve_social_public_media_url(stored)` — **separate** from WhatsApp's
`_resolve_public_media_url` (which stays exactly as-is):

```
1. stored already absolute (https://…)   → return as-is              # DO Spaces — never rewritten
2. LOCAL_MEDIA_BASE_URL set              → "{LOCAL_MEDIA_BASE_URL}/storage/{key}"   # dev/ngrok; PUBLIC_BASE_URL NOT used
3. else PUBLIC_BASE_URL set              → "{PUBLIC_BASE_URL}/storage/{key}"        # only when the override is empty
4. else                                  → None → actionable "set a public base URL" error
```

- **When `LOCAL_MEDIA_BASE_URL` is set it is used exclusively** (rule 2 short-circuits rule 3).
- **DO Spaces is never affected** — absolute URLs short-circuit at rule 1, and the override is
  purely a local-disk / dev concern, inert in any Spaces (prod) setup.

### 10.3 `LOCAL_MEDIA_BASE_URL` (local-dev only, for now)

A throwaway dev tunnel base URL so Instagram/Facebook can fetch locally-stored `/storage` images:

```
ngrok http 8000            →   https://<id>.ngrok-free.app
# .env
LOCAL_MEDIA_BASE_URL=https://<id>.ngrok-free.app
```

Upload + publish to a real IG/FB test account, no Spaces needed.

> **Ops caveat:** ngrok's free-tier browser interstitial does **not** block Meta's image fetcher
> (non-browser UA `facebookexternalhit`) — direct image fetches pass through; the interstitial
> only appears for real browsers.

## 11. Scheduling + retry poller (`src/services/social_post_poller.py`)

`social_post_poller_loop()` started in `main.py` `lifespan` next to `meta_token_monitor_loop`,
iterating every tenant DB with the **same** tenant-iteration helper. Interval
`SOCIAL_POST_POLL_INTERVAL_SECONDS` (default 60). Each tenant tick:

1. **Due scheduled** — `status=scheduled AND scheduled_at <= now()` → flip `publishing`, run `publish_post`.
2. **Retry-due targets** — `status IN (rate_limited, transient-failed) AND next_attempt_at <= now() AND attempt_count < SOCIAL_POST_MAX_ATTEMPTS` → re-run `publish_target`.
3. **Orphan media sweep** — `Media.post_id IS NULL AND created_at < now() - ORPHAN_TTL` → delete row + storage file.

**Backoff:** `next_attempt_at = now + min(SOCIAL_POST_RETRY_BASE_SECONDS * 2^(attempt-1), MAX_BACKOFF)`;
for `rate_limited`, `next_attempt_at = max(rate_limit_reset_at, now + base)`. Past
`SOCIAL_POST_MAX_ATTEMPTS` → terminal `failed` + post `needs_attention`.

The executor's `status=publishing` claim prevents double-processing; `SELECT … FOR UPDATE SKIP
LOCKED` on the poller's claim as belt-and-suspenders (single-process app, low contention).

## 12. Config additions (`src/config.py`)

Operational tunables — **no secrets**, sane defaults → **no per-tenant `.env` / provisioner
rollout required**:

| Key | Default | Purpose |
| --- | --- | --- |
| `SOCIAL_POST_POLL_INTERVAL_SECONDS` | `60` | poller tick interval |
| `SOCIAL_POST_MAX_ATTEMPTS` | `5` | retry cap per target |
| `SOCIAL_POST_RETRY_BASE_SECONDS` | `60` | backoff base |
| `SOCIAL_MAX_IMAGE_BYTES` | `8388608` (8 MB) | per-image size cap (IG limit) |
| `SOCIAL_MAX_CAROUSEL_ITEMS` | `10` | carousel cap |
| `LOCAL_MEDIA_BASE_URL` | `""` | local-dev media tunnel (ngrok); see §10 |

Reuse existing: `FACEBOOK_APP_ID` / `FACEBOOK_APP_SECRET`, `META_GRAPH_VERSION`, `PUBLIC_BASE_URL`,
plus `DO_SPACES_*` for prod storage.

## 13. Migration, permissions, session routing, Graph version

- **One alembic migration** creating `posts`, `post_targets`, `media`, `post_publish_attempts`
  → apply to dev DB (`alembic upgrade head`) **and** `scripts/migrate_all_tenants.py` (every tenant
  + `slate_template`), per root `CLAUDE.md`.
- **New `posts:publish` permission** (owner / admin / agent) added to the permission registry/roles.
- **Session routing:** all endpoints `Depends(get_tenant_scoped_db)`; the poller opens per-tenant
  sessions like the token monitor.
- **Graph version:** pin `META_GRAPH_VERSION` to a known-good version and test publish flows against
  it; on Meta deprecation, bump the one config var and re-run the publish integration tests before
  sunset.

## 14. Testing (integration, mirrors existing style — pytest + test DB)

- **Executor error mapping** — mock `MetaGraphClient` raising each `MetaGraphError`
  (190 / 80007 / 2207xxx / 5xx) → assert `error_code`, target `status`, `next_attempt_at` /
  `rate_limit_reset_at`, and a `PostPublishAttempt` row written.
- **Token-expired path** flips `account.authorized=False` + connection → `needs_reconnect`.
- **Poller** — due scheduled post → published; rate-limited target → requeued with
  `next_attempt_at == reset`; max-attempts → `failed` + `needs_attention`.
- **Endpoint validation** — IG with no media → 422; account needs reconnect → 400; oversized /
  invalid image at upload → 422; past-dated schedule → 422.
- **Happy paths** — FB single & multi-photo, IG single & carousel (mocked Graph).
- **Public-URL resolution** — Spaces absolute untouched; `LOCAL_MEDIA_BASE_URL` set bypasses
  `PUBLIC_BASE_URL`; neither set → actionable error.
- **Simulation mode** — no Graph call, `PostPublishAttempt(simulated=true)` written.

## 15. Implementation file map

**New**

- `src/models/post.py`, `src/models/post_target.py`, `src/models/media.py`, `src/models/post_publish_attempt.py`
- `src/schemas/posts.py`
- `src/services/social_publish_service.py` (executor + `classify_meta_error`)
- `src/services/social_post_poller.py` (loop)
- `src/api/posts.py` (routers)
- `src/db/repositories/posts.py` (+ targets / media / attempts repositories)
- `alembic/versions/<rev>_add_social_posts.py`
- tests under the existing integration-test layout

**Changed**

- `src/integrations/meta_graph.py` — `_post()`, publish methods, richer `MetaGraphError`
- `src/services/storage.py` — `SpacesStorageService.upload_file(..., content_type="application/pdf")`
- `src/utils/` — `resolve_social_public_media_url()` helper (+ `LOCAL_MEDIA_BASE_URL` use)
- `src/config.py` — config keys in §12
- `src/constants.py` — enums + Meta error-code constants
- `src/main.py` — start `social_post_poller_loop` in `lifespan`
- permission registry — `posts:publish`

## 16. Open items for the implementation plan

- Confirm the exact Meta token decrypt/read helper to reuse from the DEV-170 meta service.
- Confirm the `posts:publish` permission wiring matches the existing `require_permission` registry shape.
- Decide IG container status-poll bounds (max polls / interval) for image containers.
- Confirm the tenant-iteration helper exported by `meta_token_monitor` is reusable as-is by the poller.
