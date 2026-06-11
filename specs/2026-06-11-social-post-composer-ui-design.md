# Social Post Composer UI — Design

**Date:** 2026-06-11
**Scope:** Frontend only. The publish/schedule/media-upload HTTP APIs and the underlying services are already shipped on branch `feature/dev-172-api-publish-post-to-facebook-instagram-via-graph-api`.
**Companion spec:** `2026-06-09-social-publishing-facebook-instagram-design.md` (backend).

## 1. Goal

Give agency users a single page to write a caption, attach one image or video, choose which Meta platforms (Facebook Page, Instagram) to publish to, and either publish immediately or schedule the post for later. The composer must respect connection health (disable disconnected platforms) and route every post through the agency's **primary** Page / IG account for each platform.

## 2. Out of scope (v1)

- The **Posts** tab contents (list of scheduled / published / failed posts). The tab itself ships as a stub so the navigation stays consistent.
- Live Facebook / Instagram post preview. The right-column Preview card ships with a placeholder body.
- Multi-image carousels.
- Multi-video, multi-file albums.
- Draft persistence — composer state is discarded on navigation or refresh.
- Editing a scheduled post (cancel + recreate is the v1 path).
- Selecting which FB Page or IG account to post to when an agency has more than one — v1 always uses the primary.
- Orphaned-media cleanup (uploaded but never published). Deferred to a sweep job.

## 3. Route, sidebar, page shell

### Route

New route at `/social-media`, agency-visible.

Three-place registration (per `CLAUDE.md`):

1. `frontend/src/App.tsx` — `<Route path="/social-media" element={<SocialPostsPage />} />` inside the `<Layout>` block.
2. `frontend/src/components/layout/Sidebar.tsx` — `NAV_ITEMS` entry under the existing **Automation** group, label `Social Media`, `clientApp: true`, icon Lucide `Share2` (confirm against existing Automation icons at implementation time).
3. `frontend/src/components/auth/ClientModeGuard.tsx` — `CLIENT_ROUTES.add('/social-media')`.

### Page shell — `SocialPostsPage` (container)

- Header: pill tag "Social Media", title "Social posts", subtitle "Publish and schedule posts to Facebook and Instagram from one place."
- Tab strip: **Compose** | **Posts**. Tab state held in `useState<'compose' | 'posts'>`; no URL synchronisation in v1.
- Body switches between `<ComposeTab />` (this spec) and `<PostsTabPlaceholder />` (a small empty-state stub).
- Reuses the existing layout chrome; no new global styles.

The container is the only component that issues queries; everything below is presentational.

## 4. Account health gating

`SocialPostsPage` fetches the connection via the existing query key `['meta-connection']` (same cache that `MetaSettingsPage` uses) so navigating between the two pages doesn't refetch.

A pure selector derives platform health using the existing `deriveMetaHealth` helper (`frontend/src/lib/metaHealth.ts`):

```ts
type PlatformHealth = 'connected' | 'unhealthy' | 'disconnected';
type Health = { facebook: PlatformHealth; instagram: PlatformHealth };
```

`'unhealthy'` covers expired tokens or any other not-connected-but-row-exists state.

| Both disconnected | One connected, one disconnected | One connected, one unhealthy |
| ----------------- | ------------------------------- | ---------------------------- |
| Yellow banner above the composer: "No social accounts connected yet — connect Facebook or Instagram to start posting." with a `Connect accounts →` button linking to the existing `MetaSettingsPage`. All channel chips disabled. The composer still renders so users see the form. | No banner. The disconnected chip is rendered greyed-out with a tooltip "Reconnect required" and checkbox disabled. | No banner. The unhealthy chip is rendered greyed-out with tooltip "Token expired — reconnect required" and checkbox disabled. |

Disabled chips cannot be toggled on. Only `'connected'` platforms participate in submission.

## 5. ComposeTab — layout

Two-column layout on desktop (`lg:grid lg:grid-cols-3 gap-6`), stacks on mobile.

- **Left card (`col-span-2`) — `<ComposeCard>`**
  - "Publish to" row: two `<ChannelChip>` instances (Facebook, Instagram).
  - Helper text under the chips: "Disconnected platforms are greyed out — connect them in the Accounts tab."
  - Caption section: `<CaptionField>` with live char counter and hashtag highlighting.
  - Media section: `<MediaDropZone>` with drag-and-drop or file-picker fallback.
  - Publish controls: `<PublishControls>` — "Publish now" primary button and "Schedule" secondary button on the same row.
- **Right card (`col-span-1`) — `<PreviewCard>`**
  - Heading "Preview".
  - FB / IG inner tabs (visual only — not wired in v1).
  - Body: `<PreviewPlaceholder>` with copy "Live preview coming soon" and a subtle illustration. No live rendering of the caption / media.

## 6. ComposeTab — state

Single `useState` bundle in `ComposeTab`. No form library (`useState` + manual validation, per frontend rules).

```ts
type ComposeFormState = {
  caption: string;
  selectedPlatforms: { facebook: boolean; instagram: boolean };
  media: UploadedMediaFile | null;
  mediaUploadState: 'idle' | 'uploading' | 'error';
  mediaUploadError: string | null;
};
```

Schedule fields (`scheduledAt`, `timezone`) live inside the Schedule modal's own state, not in the compose bundle.

`UploadedMediaFile` is a shared type (`frontend/src/types/media.ts`):

```ts
type UploadedMediaFile = {
  id: string;
  url: string;
  kind: 'image' | 'video';
  width: number | null;
  height: number | null;
  sizeBytes: number;
};
```

## 7. Sub-components

| Component | File | Responsibilities |
| --- | --- | --- |
| `ChannelChip` | `components/socialPosts/ChannelChip.tsx` | Render one platform chip (icon + name + checkbox). Disabled state + tooltip when `health !== 'connected'`. Props: `platform`, `health`, `selected`, `onToggle`. |
| `CaptionField` | `components/socialPosts/CaptionField.tsx` | Textarea, live char counter, hashtag overlay, soft-warning row. Counter denominator = max allowed length for the selected platforms (see §8). |
| `HashtagHighlightOverlay` | `components/socialPosts/HashtagHighlightOverlay.tsx` | Internal helper used by `CaptionField` — transparent textarea over a colourised `<pre>` mirror that highlights `#hashtag` substrings. No props beyond the current caption value. |
| `MediaDropZone` | `components/media/MediaDropZone.tsx` | **Reusable** — generic drop-and-upload component. See §10. |
| `PublishControls` | `components/socialPosts/PublishControls.tsx` | "Publish now" + "Schedule" buttons. Props: `canPublish`, `isPublishing`, `onPublishClick`, `onScheduleClick`. |
| `PreviewCard` | `components/socialPosts/PreviewCard.tsx` | Right-column placeholder card with FB/IG tabs and "Live preview coming soon" body. |
| `ConfirmPublishModal` | `components/socialPosts/ConfirmPublishModal.tsx` | Small modal: title "Publish to Facebook + Instagram now?" (channel names match the selection). Body shows only the channel icons next to their names — no caption preview, no media chrome. Buttons: Cancel / Publish now. |
| `ScheduleModal` | `components/socialPosts/ScheduleModal.tsx` | Date + time + timezone pickers, "Local time: …" preview line, Cancel / Schedule buttons. See §11. |
| `AccountsConnectBanner` | `components/socialPosts/AccountsConnectBanner.tsx` | Yellow banner with "Connect accounts →" link. Rendered by `SocialPostsPage` (container), not by `ComposeTab`. |
| `PostsTabPlaceholder` | `components/socialPosts/PostsTabPlaceholder.tsx` | Empty-state stub for the Posts tab. |
| `Modal` | `components/common/Modal.tsx` | **Only create if** the codebase has no shared modal primitive. Confirm at implementation time. |

Every component is presentational — no `useQuery` / `useMutation` inside them. Data + mutations live in `ComposeTab` (the orchestrator) and `SocialPostsPage` (the container).

## 8. Validation rules

All rules live in `frontend/src/lib/socialPostValidation.ts`:

```ts
type ValidationResult = { ok: true } | { ok: false; errors: ValidationError[] };
type ValidationError = { field: 'platforms' | 'caption' | 'media' | 'accounts'; message: string; severity: 'error' | 'warning' };

function validateCompose(form: ComposeFormState, health: Health): ValidationResult;
function resolveAccountIds(connection: MetaConnection, selectedPlatforms: { facebook: boolean; instagram: boolean }): { accountIds: string[]; missingPrimary: ('facebook' | 'instagram')[] };
```

| # | Rule | Severity | Trigger |
| --- | --- | --- | --- |
| 1 | At least one platform selected | error | Submit |
| 2 | Each selected platform must be `connected` | error | Submit (gating prevents this; defensive) |
| 3 | Caption non-empty OR media attached | error | Submit |
| 4 | Caption length ≤ 2200 when Instagram selected | error | Submit (IG hard cap) |
| 5 | Caption length ≤ 63206 always | error | Submit (FB hard cap) |
| 6 | Caption length > 2200 (any selection) | warning | Live — surfaced under counter when caption crosses the threshold. Copy adapts: "Caption exceeds Instagram's 2,200-char limit." |
| 7 | Instagram selected ⇒ media required | error | Submit (IG cannot post caption-only) |
| 8 | Instagram selected, image aspect ratio < 4:5 (height/width > 1.25 OR width/height > 1.91) | warning | Upload time (after media metadata returns) |
| 9 | Image ≤ 100 MB | error | Drop / pick (client-side, before upload fires) |
| 10 | Video ≤ 1 GB | error | Drop / pick |
| 11 | A primary account exists for each selected platform | error | Submit (uses `resolveAccountIds`) |

Hard errors disable the publish/schedule buttons. Soft warnings render under the relevant field but don't block submit. All rules are unit-tested.

## 9. API integration

### Client (`frontend/src/api/posts.ts` — new)

```ts
function uploadPostMedia(file: File): Promise<MediaResponse>;             // POST /api/posts/media (multipart)
function publishPostNow(body: PostPublishRequest): Promise<PostResponse>;  // POST /api/posts/publish
function schedulePost(body: PostScheduleRequest): Promise<PostResponse>;   // POST /api/posts/schedule
```

Multipart upload uses `fetch` directly (per frontend rules — JSON helper can't carry multipart bodies). Other calls use the shared JSON helper.

### Hooks (`frontend/src/hooks/useSocialPosts.ts` — new)

```ts
function useUploadPostMedia();   // useMutation<MediaResponse, Error, File>
function usePublishPostNow();    // useMutation<PostResponse, Error, PostPublishRequest>; invalidates ['posts']
function useSchedulePost();      // useMutation<PostResponse, Error, PostScheduleRequest>; invalidates ['posts']
function useMetaConnection();    // thin re-export of the existing query
```

Each hook owns one concern (per the SRP rule for hooks). Mutations call `queryClient.invalidateQueries({ queryKey: ['posts'] })` in `onSuccess` so the future Posts tab refreshes.

### Types (`frontend/src/types/posts.ts` — new, mirrors `src/schemas/posts.py`)

```ts
type MediaResponse = { id: string; url: string; kind: 'image' | 'video'; width: number | null; height: number | null; size_bytes: number };
type PostPublishRequest = { content: string; media_ids: string[]; account_ids: string[] };
type PostScheduleRequest = PostPublishRequest & { scheduled_at: string };  // ISO 8601, UTC
type PostResponse = { id: string; status: PostStatus; scheduled_at: string | null; targets: PostTargetResponse[]; created_at: string };
type PostTargetResponse = { platform: 'facebook' | 'instagram'; status: PostTargetStatus; provider_post_id: string | null; error_code: string | null; error_message: string | null };
```

Exact field names verified against `src/schemas/posts.py` at implementation time.

## 10. MediaDropZone — reusable

Lives at `frontend/src/components/media/MediaDropZone.tsx`. Generic enough to be reused by future callers (e.g. listing photos, agent headshots) — caller injects the upload function and accepted kinds.

```ts
type MediaDropZoneProps = {
  value: UploadedMediaFile | null;
  onChange: (next: UploadedMediaFile | null) => void;
  accept: { image?: boolean; video?: boolean };
  maxImageBytes?: number;
  maxVideoBytes?: number;
  uploadFn: (file: File) => Promise<UploadedMediaFile>;
  state: 'idle' | 'uploading' | 'error';
  errorMessage?: string | null;
  label?: string;
};
```

Behaviour:

- Empty state: dashed-border drop area with copy "Drag an image or video here, or browse" + a hidden `<input type="file">` triggered by an in-area button.
- Drag-over state: solid border, primary tint background.
- Uploading: spinner overlay; drop area frozen.
- Populated: thumbnail (image) or video-poster (video) with filename + size + ✕ Remove button. Clicking ✕ calls `onChange(null)` and **does not** delete the server media (orphan cleanup is deferred).
- Validation: file-size and kind checks run **before** `uploadFn` is invoked.

Composer wires `uploadFn` to `useUploadPostMedia().mutateAsync` and translates the returned `MediaResponse` into `UploadedMediaFile`.

## 11. Submission flows

### Publish-now

1. User clicks **Publish now**. `validateCompose` runs. If hard errors → focus the first errored field, do not open the modal.
2. `<ConfirmPublishModal>` opens. Body shows only:
   - "Publish to **Facebook + Instagram** now?" (channel names reflect selection)
   - Cancel / Publish now buttons.
3. Confirm → `usePublishPostNow().mutate({ content, media_ids, account_ids })`. `account_ids` produced by `resolveAccountIds`.
4. Success → close modal, show toast "Post published", reset `ComposeFormState` to initial.
5. Failure → modal stays open, inline error message rendered above the buttons. User can retry or cancel. See §12.

### Schedule

1. User clicks **Schedule**. `validateCompose` runs. Same fail-fast.
2. `<ScheduleModal>` opens. Fields:
   - Date — `<input type="date">`, native picker.
   - Time — `<input type="time" step={300}>` (5-minute steps).
   - Timezone — `<select>` initialised to `Intl.DateTimeFormat().resolvedOptions().timeZone`. Options sourced from `Intl.supportedValuesOf('timeZone')` with a graceful fallback list (e.g. major IANA zones) if the runtime doesn't support that API.
   - Read-only "Local time: <…>" preview line showing the chosen instant in the user's browser TZ if it differs from the selected TZ.
3. Validation: the chosen instant must be **≥ 5 minutes in the future** (consistent with Meta Graph's scheduled-post requirements). Past or near-future times disable the Schedule button with inline helper text.
4. Confirm → convert `{date, time, tz}` to UTC ISO via `Intl.DateTimeFormat` + manual offset compute (no new date library; confirm whether the codebase already uses Luxon / date-fns at implementation time and reuse if so). Call `useSchedulePost().mutate(...)`.
5. Success / failure handling matches publish-now.

### Account routing — primary only

`resolveAccountIds(connection, selectedPlatforms)`:

1. For each selected platform, find the account where `platform === <p>` AND `is_primary === true`.
2. If a primary exists → use its `id`.
3. If no primary exists for a selected platform → return a hard validation error: "No primary <Facebook Page / Instagram account> set — set one in Meta settings." This blocks submit.
4. Defensive: if more than one primary appears for a platform (backend invariant violation), use the first and log a warning to `console.warn`.

The composer never exposes a page picker. Multi-page support is additive in a future spec.

## 12. Error handling

| Failure | UX |
| ------- | -- |
| Network failure / 5xx | Toast "Couldn't publish — try again." Inline message in the modal. Modal stays open. |
| 400 `SocialPublishValidationError` | Render the `detail.message` inline above the modal buttons. |
| 401 / 403 with token-expired error code | Modal swaps content: "Your Facebook token expired. Reconnect to publish." + a link to `MetaSettingsPage`. Close modal afterwards; refetch `['meta-connection']`. |
| Media upload failure | `MediaDropZone` shows `state='error'` and `errorMessage`. Drop area returns to empty after the user clicks Remove. |

No retries automatic — the user retries explicitly.

## 13. File layout

```
frontend/src/
├── api/
│   └── posts.ts                                      (new)
├── hooks/
│   └── useSocialPosts.ts                             (new)
├── lib/
│   ├── socialPostValidation.ts                       (new)
│   └── socialPostValidation.test.ts                  (new)
├── types/
│   ├── media.ts                                      (new)
│   └── posts.ts                                      (new)
├── components/
│   ├── media/
│   │   ├── MediaDropZone.tsx                         (new, reusable)
│   │   └── MediaDropZone.test.tsx                    (new)
│   ├── socialPosts/
│   │   ├── ComposeTab.tsx                            (new)
│   │   ├── ChannelChip.tsx                           (new)
│   │   ├── CaptionField.tsx                          (new)
│   │   ├── HashtagHighlightOverlay.tsx               (new)
│   │   ├── PublishControls.tsx                       (new)
│   │   ├── PreviewCard.tsx                           (new)
│   │   ├── ConfirmPublishModal.tsx                   (new)
│   │   ├── ScheduleModal.tsx                         (new)
│   │   ├── AccountsConnectBanner.tsx                 (new)
│   │   ├── PostsTabPlaceholder.tsx                   (new)
│   │   └── *.test.tsx                                (new per component above)
│   └── common/
│       └── Modal.tsx                                 (new — only if no shared modal exists)
└── pages/
    └── SocialPostsPage.tsx                           (new)
```

Files touched:

- `frontend/src/App.tsx` — add `/social-media` route.
- `frontend/src/components/layout/Sidebar.tsx` — add NAV entry under Automation, `clientApp: true`.
- `frontend/src/components/auth/ClientModeGuard.tsx` — add `/social-media` to `CLIENT_ROUTES`.
- `frontend/src/types/index.ts` — re-export new types from `posts.ts` and `media.ts`.

## 14. Testing

Unit / component tests next to source (existing convention):

- `socialPostValidation.test.ts` — every rule in §8 table-driven, plus `resolveAccountIds` (primary present, missing, duplicate).
- `MediaDropZone.test.tsx` — drop event handling, accept-filter, size-error states, upload mock.
- `CaptionField.test.tsx` — counter denominator switches per selected platforms, soft warning appears at >2200 chars, hashtag overlay tokenises correctly.
- `ChannelChip.test.tsx` — disabled state + tooltip for `disconnected` and `unhealthy`.
- `ScheduleModal.test.tsx` — TZ conversion produces correct UTC ISO, rejects past times, rejects <5 min in future.
- `ConfirmPublishModal.test.tsx` — fires mutation on confirm, surfaces server error inline.
- `SocialPostsPage.test.tsx` — banner gating logic for the four health combinations.

Manual smoke checks (because the rule "type checking and test suites verify code correctness, not feature correctness" applies):

- Run the dev server, upload an image, publish-now to a sandbox tenant. Observe success toast + post row.
- Schedule a post 10 minutes ahead in a non-local timezone. Verify the `scheduled_at` written to the DB matches the picked instant in UTC.
- Toggle off the connected platforms in `MetaSettingsPage`. Verify the banner appears and chips disable.

## 15. Open items for the implementer

- Confirm whether the codebase already has a shared modal primitive in `components/common/` before creating one.
- Confirm whether a date library (Luxon / date-fns) is already in use for TZ conversion; if not, prefer `Intl.DateTimeFormat` + manual offset compute over adding a dependency.
- Confirm the Lucide icon used for the Automation sidebar group so the Social Media entry matches visually.
- Verify the exact field names in `src/schemas/posts.py` before finalising `frontend/src/types/posts.ts`.
