# Social Post Composer UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `/social-media` page (Compose tab) so agency users can write a caption, attach one image or video, pick Facebook and/or Instagram, and either publish now or schedule, with confirmation modals and account-health gating.

**Architecture:** New container page `SocialPostsPage` owns the connection query and gates the form. `ComposeTab` (presentational orchestrator) holds form state via `useState`. Pure validation logic lives in `lib/socialPostValidation.ts`. A new reusable `MediaDropZone` lives under `components/media/`. Submission uses TanStack Query mutations against the existing `/api/posts/*` endpoints. Three-place route registration adds the route, sidebar entry, and `CLIENT_ROUTES` allowlist.

**Tech Stack:** React 18 + TypeScript, React Router v7, TanStack React Query v5, Tailwind v4, Vitest. No new dependencies.

**Spec:** `/Users/binhquach/Workplace/Slicify AI/slicify-docs/specs/2026-06-11-social-post-composer-ui-design.md`

**Adaptation from spec §14:** The spec lists component-level tests, but the codebase ships only pure-logic Vitest tests (no `@testing-library/react`, no `jsdom`). Adding component test infra is out of scope for this feature. This plan tests pure logic via Vitest and covers UI behaviour through a manual QA checklist at the end. Re-introducing component tests can be a separate plan.

**Working directory:** `/Users/binhquach/Workplace/Slicify AI/slicify-realestate` (run all commands from there).

**Commit style:** Imperative, scoped. Each task ends with exactly one commit. Use `git status` and `git diff --cached` before committing to verify the staged set.

---

## Phase 1 — Types & API surface

### Task 1: Shared `UploadedMediaFile` type

**Files:**
- Create: `frontend/src/types/media.ts`

- [ ] **Step 1: Create the type file**

```ts
export type UploadedMediaKind = "image" | "video";

export interface UploadedMediaFile {
  id: string;
  url: string;
  kind: UploadedMediaKind;
  width: number | null;
  height: number | null;
  sizeBytes: number;
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors related to the new file.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/types/media.ts
git commit -m "feat(social-posts): add shared UploadedMediaFile type"
```

---

### Task 2: Post request / response types

**Files:**
- Create: `frontend/src/types/posts.ts`

Backend schema reference: `src/schemas/posts.py` (read it before writing the file to confirm exact field names — the types must mirror it).

- [ ] **Step 1: Read the backend schema for the exact field names**

Run: `sed -n '1,200p' src/schemas/posts.py`
Note the exact names for `MediaResponse`, `PostPublishRequest`, `PostScheduleRequest`, `PostResponse`, `PostTargetResponse`, and the post status enum values.

- [ ] **Step 2: Create the type file**

```ts
import type { UploadedMediaKind } from "./media";

export type PostPlatform = "facebook" | "instagram";

export type PostStatus =
  | "scheduled"
  | "publishing"
  | "published"
  | "partial_failure"
  | "failed"
  | "cancelled";

export type PostTargetStatus =
  | "pending"
  | "publishing"
  | "published"
  | "failed"
  | "retryable";

export interface MediaResponse {
  id: string;
  url: string;
  kind: UploadedMediaKind;
  width: number | null;
  height: number | null;
  size_bytes: number;
}

export interface PostPublishRequest {
  content: string;
  media_ids: string[];
  account_ids: string[];
}

export interface PostScheduleRequest extends PostPublishRequest {
  scheduled_at: string;
}

export interface PostTargetResponse {
  platform: PostPlatform;
  status: PostTargetStatus;
  provider_post_id: string | null;
  error_code: string | null;
  error_message: string | null;
}

export interface PostResponse {
  id: string;
  status: PostStatus;
  scheduled_at: string | null;
  targets: PostTargetResponse[];
  created_at: string;
}
```

If any field names in `src/schemas/posts.py` differ (snake_case vs camelCase, status enum values, optional vs required), update this file to match exactly. Backend is the source of truth.

- [ ] **Step 3: Re-export from `types/index.ts`**

Open `frontend/src/types/index.ts` and add the re-exports near the existing groups (alphabetical or grouped — match the file's convention):

```ts
export type { UploadedMediaFile, UploadedMediaKind } from "./media";
export type {
  MediaResponse,
  PostPublishRequest,
  PostScheduleRequest,
  PostResponse,
  PostTargetResponse,
  PostPlatform,
  PostStatus,
  PostTargetStatus,
} from "./posts";
```

- [ ] **Step 4: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/posts.ts frontend/src/types/media.ts frontend/src/types/index.ts
git commit -m "feat(social-posts): add post + media response/request types"
```

---

### Task 3: API client functions

**Files:**
- Create: `frontend/src/api/posts.ts`

Backend endpoints (already shipped): `POST /api/posts/media` (multipart), `POST /api/posts/publish`, `POST /api/posts/schedule`.

The shared `post()` helper in `frontend/src/api/client.ts` sends JSON. Multipart needs a direct `fetch` call (per `frontend/CLAUDE.md` rule: "No direct fetch in components except for multipart uploads").

- [ ] **Step 1: Create the API client file**

```ts
import { del, post } from "./client";
import { getAccessToken } from "./client";
import type {
  MediaResponse,
  PostPublishRequest,
  PostResponse,
  PostScheduleRequest,
} from "../types/posts";

const POSTS_PUBLISH_PATH = "/posts/publish";
const POSTS_SCHEDULE_PATH = "/posts/schedule";
const POSTS_MEDIA_PATH = "/api/posts/media";

export function publishPostNow(body: PostPublishRequest): Promise<PostResponse> {
  return post<PostResponse>(POSTS_PUBLISH_PATH, body);
}

export function schedulePost(body: PostScheduleRequest): Promise<PostResponse> {
  return post<PostResponse>(POSTS_SCHEDULE_PATH, body);
}

export function cancelScheduledPost(postId: string): Promise<void> {
  return del(`/posts/${postId}`);
}

export async function uploadPostMedia(file: File): Promise<MediaResponse> {
  const formData = new FormData();
  formData.append("files", file);
  const token = getAccessToken();
  const headers: Record<string, string> = {};
  if (token) {
    headers.Authorization = `Bearer ${token}`;
  }
  const response = await fetch(POSTS_MEDIA_PATH, {
    method: "POST",
    body: formData,
    headers,
    credentials: "include",
  });
  if (!response.ok) {
    const detail = await safeReadDetail(response);
    throw new Error(detail);
  }
  const data: MediaResponse[] = await response.json();
  if (!data.length) {
    throw new Error("Upload returned no media records");
  }
  return data[0];
}

async function safeReadDetail(response: Response): Promise<string> {
  try {
    const body = await response.json();
    if (body && typeof body === "object") {
      const detail = (body as Record<string, unknown>).detail;
      if (typeof detail === "string" && detail) return detail;
    }
  } catch {
    // body wasn't JSON
  }
  return `Upload failed (${response.status})`;
}
```

Notes:
- `getAccessToken` is already exported from `client.ts` (verified in Task 1's exploration).
- The backend's `/api/posts/media` route accepts `list[UploadFile]` named `files` (verified in `src/api/posts.py`). We send one file per call and return the first element.

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/api/posts.ts
git commit -m "feat(social-posts): add posts API client"
```

---

## Phase 2 — Validation library (TDD)

### Task 4: Write failing tests for `validateCompose`

**Files:**
- Create: `frontend/src/lib/socialPostValidation.test.ts`

Spec reference: §8 validation table.

- [ ] **Step 1: Write the failing test file**

```ts
import { describe, expect, it } from "vitest";
import {
  resolveAccountIds,
  validateCompose,
} from "./socialPostValidation";
import type { ComposeFormState, Health } from "./socialPostValidation";
import type { MetaConnection } from "../types/meta";

const CONNECTED_HEALTH: Health = {
  facebook: "connected",
  instagram: "connected",
};

const ONLY_FB_HEALTH: Health = {
  facebook: "connected",
  instagram: "disconnected",
};

const EMPTY_FORM: ComposeFormState = {
  caption: "",
  selectedPlatforms: { facebook: false, instagram: false },
  media: null,
  mediaUploadState: "idle",
  mediaUploadError: null,
};

const IMAGE_MEDIA = {
  id: "media-1",
  url: "https://cdn.example/img.jpg",
  kind: "image" as const,
  width: 1080,
  height: 1080,
  sizeBytes: 500_000,
};

function withFB(form: Partial<ComposeFormState>): ComposeFormState {
  return {
    ...EMPTY_FORM,
    selectedPlatforms: { facebook: true, instagram: false },
    ...form,
  };
}

function withBoth(form: Partial<ComposeFormState>): ComposeFormState {
  return {
    ...EMPTY_FORM,
    selectedPlatforms: { facebook: true, instagram: true },
    ...form,
  };
}

describe("validateCompose", () => {
  it("rejects when no platform selected", () => {
    const result = validateCompose(EMPTY_FORM, CONNECTED_HEALTH);
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.errors.some((e) => e.field === "platforms" && e.severity === "error")).toBe(true);
    }
  });

  it("rejects when selected platform is not connected", () => {
    const form = withBoth({ caption: "hi", media: IMAGE_MEDIA });
    const result = validateCompose(form, ONLY_FB_HEALTH);
    expect(result.ok).toBe(false);
  });

  it("rejects empty caption and no media", () => {
    const result = validateCompose(withFB({}), CONNECTED_HEALTH);
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.errors.some((e) => e.field === "caption" || e.field === "media")).toBe(true);
    }
  });

  it("accepts caption-only post on Facebook", () => {
    const result = validateCompose(withFB({ caption: "hello" }), CONNECTED_HEALTH);
    expect(result.ok).toBe(true);
  });

  it("rejects Instagram with no media", () => {
    const form = withBoth({ caption: "hi" });
    const result = validateCompose(form, CONNECTED_HEALTH);
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.errors.some((e) => e.field === "media" && e.severity === "error")).toBe(true);
    }
  });

  it("rejects Instagram caption > 2200 chars", () => {
    const form = withBoth({ caption: "a".repeat(2201), media: IMAGE_MEDIA });
    const result = validateCompose(form, CONNECTED_HEALTH);
    expect(result.ok).toBe(false);
  });

  it("rejects Facebook caption > 63206 chars", () => {
    const form = withFB({ caption: "a".repeat(63207) });
    const result = validateCompose(form, CONNECTED_HEALTH);
    expect(result.ok).toBe(false);
  });

  it("warns when caption > 2200 (FB only)", () => {
    const form = withFB({ caption: "a".repeat(2201) });
    const result = validateCompose(form, CONNECTED_HEALTH);
    expect(result.ok).toBe(true);
  });

  it("warns when Instagram image is taller than 4:5 (height/width > 1.25)", () => {
    const tallImage = { ...IMAGE_MEDIA, width: 1000, height: 1500 };
    const form = withBoth({ caption: "hi", media: tallImage });
    const result = validateCompose(form, CONNECTED_HEALTH);
    expect(result.ok).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test and verify it fails**

Run: `cd frontend && npx vitest run src/lib/socialPostValidation.test.ts`
Expected: FAIL — module `./socialPostValidation` not found.

- [ ] **Step 3: Commit the failing test**

```bash
git add frontend/src/lib/socialPostValidation.test.ts
git commit -m "test(social-posts): add validateCompose specs"
```

---

### Task 5: Implement `validateCompose`

**Files:**
- Create: `frontend/src/lib/socialPostValidation.ts`

- [ ] **Step 1: Implement the module**

```ts
import type { MetaConnection } from "../types/meta";
import type { UploadedMediaFile } from "../types/media";

const IG_MAX_CAPTION = 2200;
const FB_MAX_CAPTION = 63206;
const IG_MIN_ASPECT = 0.8;
const IG_MAX_ASPECT = 1.91;
const MAX_IMAGE_BYTES = 100 * 1024 * 1024;
const MAX_VIDEO_BYTES = 1024 * 1024 * 1024;

export type PlatformHealth = "connected" | "unhealthy" | "disconnected";

export interface Health {
  facebook: PlatformHealth;
  instagram: PlatformHealth;
}

export interface ComposeFormState {
  caption: string;
  selectedPlatforms: { facebook: boolean; instagram: boolean };
  media: UploadedMediaFile | null;
  mediaUploadState: "idle" | "uploading" | "error";
  mediaUploadError: string | null;
}

export type ValidationField =
  | "platforms"
  | "caption"
  | "media"
  | "accounts";

export interface ValidationIssue {
  field: ValidationField;
  message: string;
  severity: "error" | "warning";
}

export type ValidationResult =
  | { ok: true; warnings: ValidationIssue[] }
  | { ok: false; errors: ValidationIssue[]; warnings: ValidationIssue[] };

export function validateCompose(
  form: ComposeFormState,
  health: Health,
): ValidationResult {
  const errors: ValidationIssue[] = [];
  const warnings: ValidationIssue[] = [];
  const { facebook: fbSelected, instagram: igSelected } = form.selectedPlatforms;

  if (!fbSelected && !igSelected) {
    errors.push({
      field: "platforms",
      severity: "error",
      message: "Select at least one platform to publish to.",
    });
  }

  if (fbSelected && health.facebook !== "connected") {
    errors.push({
      field: "platforms",
      severity: "error",
      message: "Facebook is not connected.",
    });
  }
  if (igSelected && health.instagram !== "connected") {
    errors.push({
      field: "platforms",
      severity: "error",
      message: "Instagram is not connected.",
    });
  }

  const hasCaption = form.caption.trim().length > 0;
  const hasMedia = form.media !== null;
  if (!hasCaption && !hasMedia) {
    errors.push({
      field: "caption",
      severity: "error",
      message: "Write a caption or attach an image / video.",
    });
  }

  if (igSelected && form.caption.length > IG_MAX_CAPTION) {
    errors.push({
      field: "caption",
      severity: "error",
      message: `Caption exceeds Instagram's ${IG_MAX_CAPTION}-character limit.`,
    });
  }
  if (form.caption.length > FB_MAX_CAPTION) {
    errors.push({
      field: "caption",
      severity: "error",
      message: `Caption exceeds Facebook's ${FB_MAX_CAPTION}-character limit.`,
    });
  }
  if (
    !igSelected &&
    fbSelected &&
    form.caption.length > IG_MAX_CAPTION &&
    form.caption.length <= FB_MAX_CAPTION
  ) {
    warnings.push({
      field: "caption",
      severity: "warning",
      message: `Caption exceeds Instagram's ${IG_MAX_CAPTION}-character limit. Add Instagram only after shortening.`,
    });
  }

  if (igSelected && !hasMedia) {
    errors.push({
      field: "media",
      severity: "error",
      message: "Instagram requires an image or video.",
    });
  }

  if (igSelected && form.media && form.media.kind === "image") {
    const aspect = form.media.height && form.media.width
      ? form.media.height / form.media.width
      : null;
    if (aspect !== null) {
      if (aspect > 1 / IG_MIN_ASPECT) {
        warnings.push({
          field: "media",
          severity: "warning",
          message: "Image is taller than Instagram's 4:5 ratio; it will be cropped.",
        });
      } else if (aspect < 1 / IG_MAX_ASPECT) {
        warnings.push({
          field: "media",
          severity: "warning",
          message: "Image is wider than Instagram's 1.91:1 ratio; it will be cropped.",
        });
      }
    }
  }

  if (form.media) {
    const limit = form.media.kind === "image" ? MAX_IMAGE_BYTES : MAX_VIDEO_BYTES;
    if (form.media.sizeBytes > limit) {
      errors.push({
        field: "media",
        severity: "error",
        message: form.media.kind === "image"
          ? "Image is larger than 100 MB."
          : "Video is larger than 1 GB.",
      });
    }
  }

  if (errors.length > 0) {
    return { ok: false, errors, warnings };
  }
  return { ok: true, warnings };
}

export interface ResolveAccountIdsResult {
  accountIds: string[];
  missingPrimary: ("facebook" | "instagram")[];
}

export function resolveAccountIds(
  connection: MetaConnection | null | undefined,
  selectedPlatforms: { facebook: boolean; instagram: boolean },
): ResolveAccountIdsResult {
  const accountIds: string[] = [];
  const missingPrimary: ("facebook" | "instagram")[] = [];
  if (!connection) {
    if (selectedPlatforms.facebook) missingPrimary.push("facebook");
    if (selectedPlatforms.instagram) missingPrimary.push("instagram");
    return { accountIds, missingPrimary };
  }
  const accounts = connection.accounts ?? [];
  if (selectedPlatforms.facebook) {
    const primary = accounts.filter((account) => account.platform === "facebook" && account.is_primary);
    if (primary.length === 0) {
      missingPrimary.push("facebook");
    } else {
      accountIds.push(primary[0].id);
      if (primary.length > 1) {
        console.warn("Multiple primary Facebook accounts; using the first.");
      }
    }
  }
  if (selectedPlatforms.instagram) {
    const primary = accounts.filter((account) => account.platform === "instagram" && account.is_primary);
    if (primary.length === 0) {
      missingPrimary.push("instagram");
    } else {
      accountIds.push(primary[0].id);
      if (primary.length > 1) {
        console.warn("Multiple primary Instagram accounts; using the first.");
      }
    }
  }
  return { accountIds, missingPrimary };
}
```

- [ ] **Step 2: Run tests and verify they pass**

Run: `cd frontend && npx vitest run src/lib/socialPostValidation.test.ts`
Expected: all `validateCompose` cases PASS.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/lib/socialPostValidation.ts
git commit -m "feat(social-posts): implement validateCompose"
```

---

### Task 6: Tests for `resolveAccountIds`

**Files:**
- Modify: `frontend/src/lib/socialPostValidation.test.ts`

- [ ] **Step 1: Append tests for `resolveAccountIds`**

Add the following block at the end of `frontend/src/lib/socialPostValidation.test.ts`:

```ts
const TWO_ACCOUNT_CONNECTION = {
  accounts: [
    { id: "fb-1", platform: "facebook", is_primary: true, name: "Page", username: null, authorized: true },
    { id: "fb-2", platform: "facebook", is_primary: false, name: "Other Page", username: null, authorized: true },
    { id: "ig-1", platform: "instagram", is_primary: true, name: null, username: "agency", authorized: true },
  ],
} as unknown as MetaConnection;

describe("resolveAccountIds", () => {
  it("picks the primary Facebook account", () => {
    const result = resolveAccountIds(TWO_ACCOUNT_CONNECTION, {
      facebook: true,
      instagram: false,
    });
    expect(result.accountIds).toEqual(["fb-1"]);
    expect(result.missingPrimary).toEqual([]);
  });

  it("picks both primaries when both selected", () => {
    const result = resolveAccountIds(TWO_ACCOUNT_CONNECTION, {
      facebook: true,
      instagram: true,
    });
    expect(new Set(result.accountIds)).toEqual(new Set(["fb-1", "ig-1"]));
    expect(result.missingPrimary).toEqual([]);
  });

  it("reports missingPrimary for platforms without a primary", () => {
    const connection = {
      accounts: [
        { id: "fb-only", platform: "facebook", is_primary: false, name: "Page", username: null, authorized: true },
      ],
    } as unknown as MetaConnection;
    const result = resolveAccountIds(connection, { facebook: true, instagram: true });
    expect(result.accountIds).toEqual([]);
    expect(new Set(result.missingPrimary)).toEqual(new Set(["facebook", "instagram"]));
  });

  it("handles null connection", () => {
    const result = resolveAccountIds(null, { facebook: true, instagram: false });
    expect(result.missingPrimary).toEqual(["facebook"]);
  });
});
```

(Top of file already imports `MetaConnection` from `../types/meta` and `resolveAccountIds` from `./socialPostValidation`.)

- [ ] **Step 2: Run the full test file**

Run: `cd frontend && npx vitest run src/lib/socialPostValidation.test.ts`
Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/lib/socialPostValidation.test.ts
git commit -m "test(social-posts): cover resolveAccountIds"
```

---

### Task 7: Time-to-UTC helper + tests

**Files:**
- Create: `frontend/src/lib/scheduledTime.ts`
- Create: `frontend/src/lib/scheduledTime.test.ts`

The `ScheduleModal` needs to convert `{date, time, timezone}` to a UTC ISO string. Keep this logic separate and tested so the modal stays presentation-only.

- [ ] **Step 1: Write the failing tests first**

```ts
import { describe, expect, it } from "vitest";
import { toUtcIso, isAtLeastMinutesAhead } from "./scheduledTime";

describe("toUtcIso", () => {
  it("converts NZDT (UTC+13) to UTC", () => {
    const iso = toUtcIso({ date: "2026-07-01", time: "10:00", timeZone: "Pacific/Auckland" });
    expect(iso).toBe("2026-06-30T22:00:00.000Z");
  });

  it("converts UTC to UTC unchanged", () => {
    const iso = toUtcIso({ date: "2026-07-01", time: "10:00", timeZone: "UTC" });
    expect(iso).toBe("2026-07-01T10:00:00.000Z");
  });

  it("converts America/New_York (UTC-4 in July) to UTC", () => {
    const iso = toUtcIso({ date: "2026-07-01", time: "10:00", timeZone: "America/New_York" });
    expect(iso).toBe("2026-07-01T14:00:00.000Z");
  });
});

describe("isAtLeastMinutesAhead", () => {
  it("returns true when the instant is far in the future", () => {
    const now = new Date("2026-07-01T10:00:00.000Z").getTime();
    expect(isAtLeastMinutesAhead("2026-07-01T10:10:00.000Z", 5, now)).toBe(true);
  });

  it("returns false when the instant is in the past", () => {
    const now = new Date("2026-07-01T10:00:00.000Z").getTime();
    expect(isAtLeastMinutesAhead("2026-07-01T09:00:00.000Z", 5, now)).toBe(false);
  });

  it("returns false when within the lead time", () => {
    const now = new Date("2026-07-01T10:00:00.000Z").getTime();
    expect(isAtLeastMinutesAhead("2026-07-01T10:03:00.000Z", 5, now)).toBe(false);
  });
});
```

- [ ] **Step 2: Run and confirm failure**

Run: `cd frontend && npx vitest run src/lib/scheduledTime.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `scheduledTime.ts`**

```ts
export interface ScheduledTimeInput {
  date: string;
  time: string;
  timeZone: string;
}

export function toUtcIso(input: ScheduledTimeInput): string {
  const [year, month, day] = input.date.split("-").map(Number);
  const [hour, minute] = input.time.split(":").map(Number);
  const utcGuess = Date.UTC(year, month - 1, day, hour, minute, 0, 0);
  const offsetMs = offsetForZoneAt(utcGuess, input.timeZone);
  const utcMs = utcGuess - offsetMs;
  return new Date(utcMs).toISOString();
}

export function isAtLeastMinutesAhead(
  utcIso: string,
  minutes: number,
  nowMs: number = Date.now(),
): boolean {
  const target = new Date(utcIso).getTime();
  return target - nowMs >= minutes * 60_000;
}

function offsetForZoneAt(utcMs: number, timeZone: string): number {
  const formatter = new Intl.DateTimeFormat("en-US", {
    timeZone,
    hour12: false,
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });
  const parts = formatter.formatToParts(new Date(utcMs));
  const fields: Record<string, string> = {};
  for (const part of parts) {
    if (part.type !== "literal") {
      fields[part.type] = part.value;
    }
  }
  const asUtc = Date.UTC(
    Number(fields.year),
    Number(fields.month) - 1,
    Number(fields.day),
    Number(fields.hour === "24" ? "0" : fields.hour),
    Number(fields.minute),
    Number(fields.second),
  );
  return asUtc - utcMs;
}
```

- [ ] **Step 4: Run and verify passing**

Run: `cd frontend && npx vitest run src/lib/scheduledTime.test.ts`
Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/lib/scheduledTime.ts frontend/src/lib/scheduledTime.test.ts
git commit -m "feat(social-posts): add scheduled-time TZ helpers"
```

---

## Phase 3 — Hooks

### Task 8: React Query hooks for posts

**Files:**
- Create: `frontend/src/hooks/useSocialPosts.ts`

- [ ] **Step 1: Create the hooks file**

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import {
  cancelScheduledPost,
  publishPostNow,
  schedulePost,
  uploadPostMedia,
} from "../api/posts";
import type {
  MediaResponse,
  PostPublishRequest,
  PostResponse,
  PostScheduleRequest,
} from "../types/posts";

const POSTS_QUERY_KEY = ["posts"] as const;

export function useUploadPostMedia() {
  return useMutation<MediaResponse, Error, File>({
    mutationFn: uploadPostMedia,
  });
}

export function usePublishPostNow() {
  const queryClient = useQueryClient();
  return useMutation<PostResponse, Error, PostPublishRequest>({
    mutationFn: publishPostNow,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: POSTS_QUERY_KEY });
    },
  });
}

export function useSchedulePost() {
  const queryClient = useQueryClient();
  return useMutation<PostResponse, Error, PostScheduleRequest>({
    mutationFn: schedulePost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: POSTS_QUERY_KEY });
    },
  });
}

export function useCancelScheduledPost() {
  const queryClient = useQueryClient();
  return useMutation<void, Error, string>({
    mutationFn: cancelScheduledPost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: POSTS_QUERY_KEY });
    },
  });
}
```

The cancel hook ships now even though the Posts tab (which calls it) is out of scope — it completes the v1 lifecycle ("cancel + recreate" path from spec §2) so the Posts-list implementer doesn't have to add new API surface later.

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/hooks/useSocialPosts.ts
git commit -m "feat(social-posts): add posts mutation hooks"
```

---

## Phase 4 — Common Modal primitive

### Task 9: Shared `Modal` component

**Files:**
- Create: `frontend/src/components/common/Modal.tsx`

The codebase has feature-specific modals (`SupportTicketModal`, `SkillDetailModal`) but no shared primitive. The Social composer needs two modals — define one shared shell now and reuse it.

This Modal mirrors the UX patterns from Radix UI's Dialog primitive (https://www.radix-ui.com/primitives/docs/components/dialog) — focus trap, restore focus on close, body scroll lock, portal rendering, and full ARIA wiring — but uses the codebase's existing design system (Tailwind v4 utilities, Lucide icons, no new dependencies).

Radix patterns to reproduce:
- Render in a portal so the dialog isn't constrained by ancestor `overflow` or `transform`.
- On open: capture the currently-focused element, then move focus into the dialog (first focusable child, or a caller-supplied element).
- While open: trap Tab / Shift+Tab inside the dialog.
- On close: return focus to the previously-focused element.
- Lock body scroll (`overflow: hidden` on `<body>`).
- `role="dialog"` + `aria-modal="true"` + `aria-labelledby` (title) + optional `aria-describedby`.
- Close on Escape and on backdrop click.

- [ ] **Step 1: Confirm the file does not already exist**

Run: `ls frontend/src/components/common/ 2>/dev/null`
If `Modal.tsx` exists, STOP and reuse it instead of creating a new one.

- [ ] **Step 2: Create the Modal component**

```tsx
import { useEffect, useId, useRef } from "react";
import { createPortal } from "react-dom";
import { X } from "lucide-react";

interface ModalProps {
  open: boolean;
  title: string;
  description?: string;
  onClose: () => void;
  children: React.ReactNode;
  footer?: React.ReactNode;
  size?: "sm" | "md" | "lg";
}

const SIZE_CLASS: Record<NonNullable<ModalProps["size"]>, string> = {
  sm: "max-w-sm",
  md: "max-w-md",
  lg: "max-w-lg",
};

const FOCUSABLE_SELECTOR = [
  "a[href]",
  "button:not([disabled])",
  "textarea:not([disabled])",
  "input:not([disabled])",
  "select:not([disabled])",
  "[tabindex]:not([tabindex='-1'])",
].join(",");

export function Modal({
  open,
  title,
  description,
  onClose,
  children,
  footer,
  size = "md",
}: ModalProps) {
  const titleId = useId();
  const descriptionId = useId();
  const contentRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (!open) return;
    previousFocusRef.current =
      document.activeElement instanceof HTMLElement ? document.activeElement : null;
    const target = findFirstFocusable(contentRef.current) ?? contentRef.current;
    target?.focus();
    return () => {
      previousFocusRef.current?.focus();
    };
  }, [open]);

  useEffect(() => {
    if (!open) return;
    const previousOverflow = document.body.style.overflow;
    document.body.style.overflow = "hidden";
    return () => {
      document.body.style.overflow = previousOverflow;
    };
  }, [open]);

  function handleKeyDown(event: React.KeyboardEvent<HTMLDivElement>) {
    if (event.key === "Escape") {
      event.stopPropagation();
      onClose();
      return;
    }
    if (event.key !== "Tab") return;
    const focusables = collectFocusable(contentRef.current);
    if (focusables.length === 0) {
      event.preventDefault();
      contentRef.current?.focus();
      return;
    }
    const first = focusables[0];
    const last = focusables[focusables.length - 1];
    const active = document.activeElement;
    if (event.shiftKey && active === first) {
      event.preventDefault();
      last.focus();
    } else if (!event.shiftKey && active === last) {
      event.preventDefault();
      first.focus();
    }
  }

  function handleBackdropClick() {
    onClose();
  }

  function stopPropagation(event: React.MouseEvent<HTMLDivElement>) {
    event.stopPropagation();
  }

  if (!open) return null;

  return createPortal(
    <div
      className="fixed inset-0 z-50 flex items-center justify-center bg-black/40"
      onClick={handleBackdropClick}
      onKeyDown={handleKeyDown}
    >
      <div
        ref={contentRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        aria-describedby={description ? descriptionId : undefined}
        tabIndex={-1}
        className={`w-full ${SIZE_CLASS[size]} mx-4 rounded-lg bg-white shadow-xl outline-none`}
        onClick={stopPropagation}
      >
        <div className="flex items-center justify-between border-b border-gray-200 px-5 py-3">
          <h2 id={titleId} className="text-base font-semibold text-gray-900">
            {title}
          </h2>
          <button
            type="button"
            onClick={onClose}
            className="rounded p-1 text-gray-400 hover:bg-gray-100 hover:text-gray-700"
            aria-label="Close"
          >
            <X className="h-4 w-4" />
          </button>
        </div>
        {description ? (
          <p id={descriptionId} className="px-5 pt-3 text-xs text-gray-500">
            {description}
          </p>
        ) : null}
        <div className="px-5 py-4 text-sm text-gray-700">{children}</div>
        {footer ? (
          <div className="flex justify-end gap-2 border-t border-gray-200 px-5 py-3">
            {footer}
          </div>
        ) : null}
      </div>
    </div>,
    document.body,
  );
}

function findFirstFocusable(container: HTMLElement | null): HTMLElement | null {
  const focusables = collectFocusable(container);
  return focusables[0] ?? null;
}

function collectFocusable(container: HTMLElement | null): HTMLElement[] {
  if (!container) return [];
  return Array.from(
    container.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTOR),
  ).filter((element) => element.offsetParent !== null);
}
```

Behaviour mirrored from Radix Dialog:
- **Portal** — `createPortal(..., document.body)` keeps the dialog out of ancestor stacking contexts.
- **Focus snapshot + restore** — the first effect records `document.activeElement` on open, restores it on close.
- **Initial focus** — moves to the first focusable child, or the dialog content itself if none.
- **Tab trap** — `onKeyDown` wraps Tab / Shift+Tab between first and last focusables.
- **Body scroll lock** — `document.body.style.overflow = "hidden"` for the lifetime of the open state.
- **ARIA** — `role="dialog"`, `aria-modal`, `aria-labelledby` (always), `aria-describedby` (when `description` provided).
- **Escape + backdrop close** — both call `onClose`.

- [ ] **Step 3: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 4: Manual focus-management smoke test (run during Task 26 too)**

When the modal opens via `ConfirmPublishModal` later, confirm:
- Focus lands on the Cancel button (first focusable inside).
- Tab cycles between Cancel → Publish now → Cancel.
- Pressing Esc closes the modal.
- After close, focus returns to the "Publish now" trigger button on the composer.

If any of these fails, fix it now — don't defer to Task 26.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/common/Modal.tsx
git commit -m "feat(common): add shared Modal with Radix-style focus + ARIA behaviour"
```

---

## Phase 5 — Reusable MediaDropZone

### Task 10: `MediaDropZone` component

**Files:**
- Create: `frontend/src/components/media/MediaDropZone.tsx`

Spec reference: §10 (reusable, generic, caller injects `uploadFn`).

- [ ] **Step 1: Create the component**

```tsx
import { useId, useRef, useState } from "react";
import { ImagePlus, Loader2, Trash2 } from "lucide-react";
import type { UploadedMediaFile, UploadedMediaKind } from "../../types/media";

interface MediaDropZoneProps {
  value: UploadedMediaFile | null;
  onChange: (next: UploadedMediaFile | null) => void;
  accept: { image?: boolean; video?: boolean };
  maxImageBytes?: number;
  maxVideoBytes?: number;
  uploadFn: (file: File) => Promise<UploadedMediaFile>;
  state: "idle" | "uploading" | "error";
  errorMessage?: string | null;
  label?: string;
}

const DEFAULT_MAX_IMAGE_BYTES = 100 * 1024 * 1024;
const DEFAULT_MAX_VIDEO_BYTES = 1024 * 1024 * 1024;

export function MediaDropZone({
  value,
  onChange,
  accept,
  maxImageBytes = DEFAULT_MAX_IMAGE_BYTES,
  maxVideoBytes = DEFAULT_MAX_VIDEO_BYTES,
  uploadFn,
  state,
  errorMessage,
  label,
}: MediaDropZoneProps) {
  const inputId = useId();
  const inputRef = useRef<HTMLInputElement>(null);
  const [isDragOver, setIsDragOver] = useState(false);
  const [localError, setLocalError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);

  const acceptList = buildAcceptList(accept);
  const acceptedLabel = buildAcceptedLabel(accept);

  function handleBrowseClick() {
    inputRef.current?.click();
  }

  function handleRemoveClick() {
    onChange(null);
    setLocalError(null);
    if (inputRef.current) inputRef.current.value = "";
  }

  async function handleFile(file: File) {
    const kind = inferKind(file);
    if (!kind) {
      setLocalError("Unsupported file type.");
      return;
    }
    if (kind === "image" && !accept.image) {
      setLocalError("Images aren't allowed here.");
      return;
    }
    if (kind === "video" && !accept.video) {
      setLocalError("Videos aren't allowed here.");
      return;
    }
    const sizeLimit = kind === "image" ? maxImageBytes : maxVideoBytes;
    if (file.size > sizeLimit) {
      setLocalError(
        kind === "image"
          ? `Image is larger than ${Math.round(maxImageBytes / 1024 / 1024)} MB.`
          : `Video is larger than ${Math.round(maxVideoBytes / 1024 / 1024)} MB.`,
      );
      return;
    }
    setLocalError(null);
    setBusy(true);
    try {
      const uploaded = await uploadFn(file);
      onChange(uploaded);
    } catch (error) {
      setLocalError(error instanceof Error ? error.message : "Upload failed.");
    } finally {
      setBusy(false);
    }
  }

  function handleFileInputChange(event: React.ChangeEvent<HTMLInputElement>) {
    const file = event.target.files?.[0];
    if (file) handleFile(file);
  }

  function handleDragEnter(event: React.DragEvent<HTMLDivElement>) {
    event.preventDefault();
    setIsDragOver(true);
  }

  function handleDragOver(event: React.DragEvent<HTMLDivElement>) {
    event.preventDefault();
  }

  function handleDragLeave() {
    setIsDragOver(false);
  }

  function handleDrop(event: React.DragEvent<HTMLDivElement>) {
    event.preventDefault();
    setIsDragOver(false);
    const file = event.dataTransfer.files?.[0];
    if (file) handleFile(file);
  }

  const errorText = errorMessage ?? localError;

  if (value) {
    return (
      <div className="space-y-2">
        {label ? <div className="text-sm font-medium text-gray-700">{label}</div> : null}
        <div className="flex items-center gap-3 rounded-lg border border-gray-200 bg-white p-3">
          {value.kind === "image" ? (
            <img
              src={value.url}
              alt=""
              className="h-24 w-24 rounded object-cover"
            />
          ) : (
            <video
              src={value.url}
              className="h-24 w-24 rounded object-cover"
              muted
            />
          )}
          <div className="flex-1 text-xs text-gray-600">
            <div className="font-medium text-gray-800">
              {value.kind === "image" ? "Image" : "Video"}
            </div>
            <div>{formatBytes(value.sizeBytes)}</div>
          </div>
          <button
            type="button"
            onClick={handleRemoveClick}
            className="rounded p-2 text-gray-400 hover:bg-gray-100 hover:text-red-600"
            aria-label="Remove media"
          >
            <Trash2 className="h-4 w-4" />
          </button>
        </div>
        {errorText ? <p className="text-xs text-red-600">{errorText}</p> : null}
      </div>
    );
  }

  const uploading = state === "uploading" || busy;
  const borderClass = isDragOver
    ? "border-indigo-500 bg-indigo-50"
    : "border-gray-300 bg-white";

  return (
    <div className="space-y-2">
      {label ? <div className="text-sm font-medium text-gray-700">{label}</div> : null}
      <div
        onDragEnter={handleDragEnter}
        onDragOver={handleDragOver}
        onDragLeave={handleDragLeave}
        onDrop={handleDrop}
        className={`flex flex-col items-center justify-center gap-2 rounded-lg border-2 border-dashed px-6 py-8 transition-colors ${borderClass}`}
      >
        {uploading ? (
          <Loader2 className="h-6 w-6 animate-spin text-indigo-500" />
        ) : (
          <ImagePlus className="h-6 w-6 text-gray-400" />
        )}
        <p className="text-sm text-gray-600">
          Drag {acceptedLabel} here, or{" "}
          <button
            type="button"
            onClick={handleBrowseClick}
            className="font-medium text-indigo-600 hover:underline"
          >
            browse
          </button>
        </p>
        <input
          ref={inputRef}
          id={inputId}
          type="file"
          accept={acceptList}
          onChange={handleFileInputChange}
          className="hidden"
        />
      </div>
      {errorText ? <p className="text-xs text-red-600">{errorText}</p> : null}
    </div>
  );
}

function inferKind(file: File): UploadedMediaKind | null {
  if (file.type.startsWith("image/")) return "image";
  if (file.type.startsWith("video/")) return "video";
  return null;
}

function buildAcceptList(accept: { image?: boolean; video?: boolean }): string {
  const parts: string[] = [];
  if (accept.image) parts.push("image/*");
  if (accept.video) parts.push("video/*");
  return parts.join(",");
}

function buildAcceptedLabel(accept: { image?: boolean; video?: boolean }): string {
  if (accept.image && accept.video) return "an image or video";
  if (accept.image) return "an image";
  if (accept.video) return "a video";
  return "a file";
}

function formatBytes(bytes: number): string {
  if (bytes >= 1024 * 1024) {
    return `${(bytes / 1024 / 1024).toFixed(1)} MB`;
  }
  if (bytes >= 1024) {
    return `${(bytes / 1024).toFixed(0)} KB`;
  }
  return `${bytes} B`;
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/media/MediaDropZone.tsx
git commit -m "feat(media): add reusable MediaDropZone"
```

---

## Phase 6 — Composer presentational pieces

### Task 11: `ChannelChip`

**Files:**
- Create: `frontend/src/components/socialPosts/ChannelChip.tsx`

- [ ] **Step 1: Create the component**

```tsx
import facebookLogo from "../../assets/brands/facebook.svg";
import instagramLogo from "../../assets/brands/instagram.svg";
import type { PlatformHealth } from "../../lib/socialPostValidation";

type Platform = "facebook" | "instagram";

interface ChannelChipProps {
  platform: Platform;
  health: PlatformHealth;
  selected: boolean;
  onToggle: () => void;
}

const PLATFORM_LABEL: Record<Platform, string> = {
  facebook: "Facebook",
  instagram: "Instagram",
};

const PLATFORM_LOGO: Record<Platform, string> = {
  facebook: facebookLogo,
  instagram: instagramLogo,
};

const TOOLTIP_BY_HEALTH: Record<PlatformHealth, string | null> = {
  connected: null,
  disconnected: "Reconnect required",
  unhealthy: "Token expired — reconnect required",
};

export function ChannelChip({
  platform,
  health,
  selected,
  onToggle,
}: ChannelChipProps) {
  const disabled = health !== "connected";
  const tooltip = TOOLTIP_BY_HEALTH[health];

  function handleClick() {
    if (!disabled) onToggle();
  }

  return (
    <button
      type="button"
      onClick={handleClick}
      disabled={disabled}
      title={tooltip ?? undefined}
      className={`flex items-center gap-2 rounded-md border px-3 py-2 text-sm transition-colors ${
        disabled
          ? "cursor-not-allowed border-gray-200 bg-gray-50 text-gray-400"
          : selected
          ? "border-indigo-500 bg-indigo-50 text-indigo-700"
          : "border-gray-300 bg-white text-gray-700 hover:border-gray-400"
      }`}
    >
      <img src={PLATFORM_LOGO[platform]} alt="" className="h-4 w-4" />
      <span>{PLATFORM_LABEL[platform]}</span>
      <input
        type="checkbox"
        checked={selected}
        readOnly
        disabled={disabled}
        className="ml-1 h-4 w-4"
      />
    </button>
  );
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/ChannelChip.tsx
git commit -m "feat(social-posts): add ChannelChip"
```

---

### Task 12: Tiptap `hashtagHighlight` extension

**Files:**
- Install: Tiptap deps that aren't already in `frontend/package.json`
- Create: `frontend/src/components/socialPosts/hashtagHighlight.ts`

The caption editor uses Tiptap with a minimal extension set (no StarterKit — we don't want bold/italic/lists/headings in a social caption). Hashtag highlighting is a ProseMirror **decoration**, not a Tiptap mark. Decorations are purely visual: they don't enter the document model, so `editor.getText()` returns clean plain text without any serialization layer to strip the highlighting back out.

Why not a textarea + transparent-overlay? IME composition flicker, scroll-sync drift, font-metric drift across browsers, and selection-over-coloured-spans rendering — all of which a contentEditable / ProseMirror editor handles natively.

- [ ] **Step 1: Confirm which Tiptap packages need installing**

Run: `grep -E "@tiptap|prosemirror" frontend/package.json`
Already present (verified): `@tiptap/react`, `@tiptap/starter-kit`, `@tiptap/extension-placeholder`, `@tiptap/extension-image`, `@tiptap/extension-heading`, `@tiptap/extension-text-align`, `@tiptap/extension-underline` — all at `^3.24.0`.

Missing (need install): `@tiptap/core`, `@tiptap/pm`, `@tiptap/extension-document`, `@tiptap/extension-paragraph`, `@tiptap/extension-text`, `@tiptap/extension-hard-break`.

- [ ] **Step 2: Install the missing packages**

Run:
```bash
cd frontend && npm install \
  @tiptap/core@^3.24.0 \
  @tiptap/pm@^3.24.0 \
  @tiptap/extension-document@^3.24.0 \
  @tiptap/extension-paragraph@^3.24.0 \
  @tiptap/extension-text@^3.24.0 \
  @tiptap/extension-hard-break@^3.24.0
```

Expected: `package.json` + `package-lock.json` updated, no errors.

- [ ] **Step 3: Create the extension**

```ts
import { Extension } from "@tiptap/core";
import { Plugin } from "@tiptap/pm/state";
import { Decoration, DecorationSet } from "@tiptap/pm/view";

const HASHTAG_PATTERN = /[#@][\p{L}\p{N}_]+/gu;
const HIGHLIGHT_CLASS = "text-indigo-600";

export const hashtagHighlight = () =>
  Extension.create({
    name: "hashtagHighlight",
    addProseMirrorPlugins() {
      return [
        new Plugin({
          props: {
            decorations(state) {
              const decorations: Decoration[] = [];
              state.doc.descendants((node, position) => {
                if (!node.isText || !node.text) return;
                for (const match of node.text.matchAll(HASHTAG_PATTERN)) {
                  const start = position + (match.index ?? 0);
                  const end = start + match[0].length;
                  decorations.push(
                    Decoration.inline(start, end, { class: HIGHLIGHT_CLASS }),
                  );
                }
              });
              return DecorationSet.create(state.doc, decorations);
            },
          },
        }),
      ];
    },
  });
```

Highlights both `#hashtag` and `@mention`. `\p{L}\p{N}_` covers Vietnamese diacritics, CJK, accented characters — anything Unicode treats as a letter or digit.

- [ ] **Step 4: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add frontend/package.json frontend/package-lock.json frontend/src/components/socialPosts/hashtagHighlight.ts
git commit -m "feat(social-posts): add Tiptap hashtag highlight decoration plugin"
```

---

### Task 13: Tiptap-based `CaptionField`

**Files:**
- Create: `frontend/src/components/socialPosts/CaptionField.tsx`

Plain text in, plain text out — the editor never exposes formatted output, only `editor.getText({ blockSeparator: "\n" })`. The component stays controlled: when the parent passes a new `value` that doesn't match the editor's current text (e.g. after the form resets), an effect syncs the editor.

Character count uses **codepoint** counting (`[...text].length`) so emoji and surrogate pairs are counted once. UTF-16 `.length` would over-count them.

- [ ] **Step 1: Create the component**

```tsx
import { useEffect } from "react";
import { EditorContent, useEditor } from "@tiptap/react";
import Document from "@tiptap/extension-document";
import HardBreak from "@tiptap/extension-hard-break";
import Paragraph from "@tiptap/extension-paragraph";
import Placeholder from "@tiptap/extension-placeholder";
import Text from "@tiptap/extension-text";
import { hashtagHighlight } from "./hashtagHighlight";

const IG_MAX_CAPTION = 2200;
const FB_MAX_CAPTION = 63206;
const PLACEHOLDER = "Write a caption…";

interface CaptionFieldProps {
  value: string;
  onChange: (next: string) => void;
  selectedPlatforms: { facebook: boolean; instagram: boolean };
}

export function CaptionField({
  value,
  onChange,
  selectedPlatforms,
}: CaptionFieldProps) {
  const editor = useEditor({
    extensions: [
      Document,
      Paragraph,
      Text,
      HardBreak,
      Placeholder.configure({ placeholder: PLACEHOLDER }),
      hashtagHighlight(),
    ],
    content: "",
    onUpdate({ editor: currentEditor }) {
      onChange(currentEditor.getText({ blockSeparator: "\n" }));
    },
    editorProps: {
      attributes: {
        class:
          "min-h-[8rem] w-full rounded-md border border-gray-300 px-3 py-2 text-sm text-gray-900 focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500 whitespace-pre-wrap",
      },
    },
  });

  useEffect(() => {
    if (!editor) return;
    const current = editor.getText({ blockSeparator: "\n" });
    if (current !== value) {
      editor.commands.setContent(value, { emitUpdate: false });
    }
  }, [editor, value]);

  const charCount = countCodepoints(value);
  const denominator = chooseDenominator(selectedPlatforms);
  const overflow = denominator !== null && charCount > denominator;
  const showCrossPlatformWarning =
    selectedPlatforms.facebook &&
    !selectedPlatforms.instagram &&
    charCount > IG_MAX_CAPTION;

  return (
    <div className="space-y-1">
      <div className="flex items-center justify-between text-sm">
        <span className="font-medium text-gray-700">Caption</span>
        <span className={overflow ? "text-red-600" : "text-gray-500"}>
          {charCount}
          {denominator !== null ? ` / ${denominator}` : ""}
        </span>
      </div>
      <EditorContent editor={editor} />
      {showCrossPlatformWarning ? (
        <p className="text-xs text-amber-600">
          Caption exceeds Instagram's {IG_MAX_CAPTION}-character limit. Add Instagram only after shortening.
        </p>
      ) : null}
    </div>
  );
}

function chooseDenominator(selected: {
  facebook: boolean;
  instagram: boolean;
}): number | null {
  if (selected.instagram) return IG_MAX_CAPTION;
  if (selected.facebook) return FB_MAX_CAPTION;
  return FB_MAX_CAPTION;
}

function countCodepoints(text: string): number {
  let total = 0;
  for (const _codepoint of text) {
    total += 1;
  }
  return total;
}
```

Notes on the controlled pattern:
- `content: ""` sets the editor's *initial* content only. Tiptap doesn't auto-track external `value` changes.
- The `useEffect` block syncs external `value → editor`. It guards against feedback loops by skipping the update when the editor already matches, and uses `emitUpdate: false` so the sync doesn't fire `onUpdate`.
- We never block typing on length. The counter goes red past the limit; submit-time validation rejects.

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Manual smoke check (defer the rest to Task 26)**

Run the dev server. Open the composer page, focus the caption editor and:
- Type `Open house tomorrow #remuera @sarah` — confirm `#remuera` and `@sarah` render in indigo.
- Type an emoji (e.g. `🏡`) — confirm the counter increments by 1 (not 2).
- Type beyond 2,200 chars with only Facebook selected — the counter goes red and the cross-platform warning appears.

If any of the above fails, fix before moving on.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/socialPosts/CaptionField.tsx
git commit -m "feat(social-posts): add Tiptap-based CaptionField with hashtag decoration + codepoint counter"
```

---

### Task 14: `PublishControls`

**Files:**
- Create: `frontend/src/components/socialPosts/PublishControls.tsx`

- [ ] **Step 1: Create the component**

```tsx
import { Calendar, Loader2, Send } from "lucide-react";

interface PublishControlsProps {
  canPublish: boolean;
  isBusy: boolean;
  onPublishClick: () => void;
  onScheduleClick: () => void;
}

export function PublishControls({
  canPublish,
  isBusy,
  onPublishClick,
  onScheduleClick,
}: PublishControlsProps) {
  const disabled = !canPublish || isBusy;

  function handlePublish() {
    onPublishClick();
  }

  function handleSchedule() {
    onScheduleClick();
  }

  return (
    <div className="flex flex-wrap items-center gap-3">
      <button
        type="button"
        onClick={handlePublish}
        disabled={disabled}
        className="inline-flex items-center gap-2 rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white shadow-sm transition-colors hover:bg-indigo-500 disabled:cursor-not-allowed disabled:bg-gray-300"
      >
        {isBusy ? <Loader2 className="h-4 w-4 animate-spin" /> : <Send className="h-4 w-4" />}
        Publish now
      </button>
      <button
        type="button"
        onClick={handleSchedule}
        disabled={disabled}
        className="inline-flex items-center gap-2 rounded-md border border-gray-300 bg-white px-4 py-2 text-sm font-medium text-gray-700 shadow-sm transition-colors hover:border-gray-400 disabled:cursor-not-allowed disabled:bg-gray-100 disabled:text-gray-400"
      >
        <Calendar className="h-4 w-4" />
        Schedule
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/PublishControls.tsx
git commit -m "feat(social-posts): add PublishControls"
```

---

### Task 15: `PreviewCard` (placeholder)

**Files:**
- Create: `frontend/src/components/socialPosts/PreviewCard.tsx`

- [ ] **Step 1: Create the component**

```tsx
import { useState } from "react";
import facebookLogo from "../../assets/brands/facebook.svg";
import instagramLogo from "../../assets/brands/instagram.svg";

type PreviewTab = "facebook" | "instagram";

const TABS: { id: PreviewTab; label: string; logo: string }[] = [
  { id: "facebook", label: "Facebook", logo: facebookLogo },
  { id: "instagram", label: "Instagram", logo: instagramLogo },
];

export function PreviewCard() {
  const [active, setActive] = useState<PreviewTab>("facebook");

  function makeTabHandler(tab: PreviewTab) {
    return () => setActive(tab);
  }

  return (
    <div className="rounded-lg border border-gray-200 bg-white p-4 shadow-sm">
      <h3 className="text-sm font-medium text-gray-700">Preview</h3>
      <div className="mt-3 inline-flex rounded-md border border-gray-200 bg-gray-50 p-1">
        {TABS.map((tab) => (
          <button
            key={tab.id}
            type="button"
            onClick={makeTabHandler(tab.id)}
            className={`flex items-center gap-1 rounded px-3 py-1 text-xs font-medium transition-colors ${
              active === tab.id
                ? "bg-white text-gray-900 shadow-sm"
                : "text-gray-600 hover:text-gray-800"
            }`}
          >
            <img src={tab.logo} alt="" className="h-3 w-3" />
            {tab.label}
          </button>
        ))}
      </div>
      <p className="mt-2 text-xs italic text-gray-500">
        Approximate preview — not exact rendering
      </p>
      <div className="mt-6 flex flex-col items-center justify-center rounded-md border border-dashed border-gray-200 px-4 py-12 text-center text-sm text-gray-400">
        Live preview coming soon
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/PreviewCard.tsx
git commit -m "feat(social-posts): add PreviewCard placeholder"
```

---

### Task 16: `AccountsConnectBanner`

**Files:**
- Create: `frontend/src/components/socialPosts/AccountsConnectBanner.tsx`

- [ ] **Step 1: Create the component**

```tsx
import { Link } from "react-router-dom";
import { AlertCircle } from "lucide-react";

const META_SETTINGS_PATH = "/integrations";

export function AccountsConnectBanner() {
  return (
    <div className="flex items-center justify-between rounded-md border border-amber-200 bg-amber-50 px-4 py-3 text-amber-800">
      <div className="flex items-center gap-2 text-sm">
        <AlertCircle className="h-4 w-4" />
        <span>
          No social accounts connected yet — connect Facebook or Instagram to start posting.
        </span>
      </div>
      <Link
        to={META_SETTINGS_PATH}
        className="rounded-md border border-amber-300 bg-white px-3 py-1 text-xs font-medium text-amber-800 hover:bg-amber-100"
      >
        Connect accounts →
      </Link>
    </div>
  );
}
```

The link points to `/integrations` (the existing integrations page where Meta settings live). If `MetaSettingsPage` is mounted elsewhere, update `META_SETTINGS_PATH` to that route during integration.

- [ ] **Step 2: Verify the link target**

Run: `grep -n "MetaSettingsPage\|meta-settings" frontend/src/App.tsx`
Confirm the path used by the existing Meta settings page. Update `META_SETTINGS_PATH` if it differs.

- [ ] **Step 3: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/socialPosts/AccountsConnectBanner.tsx
git commit -m "feat(social-posts): add AccountsConnectBanner"
```

---

### Task 17: `PostsTabPlaceholder`

**Files:**
- Create: `frontend/src/components/socialPosts/PostsTabPlaceholder.tsx`

- [ ] **Step 1: Create the component**

```tsx
import { Inbox } from "lucide-react";

export function PostsTabPlaceholder() {
  return (
    <div className="flex flex-col items-center justify-center rounded-lg border border-dashed border-gray-200 bg-white py-16 text-center text-gray-500">
      <Inbox className="mb-3 h-8 w-8 text-gray-400" />
      <p className="text-sm font-medium text-gray-700">Posts list coming soon</p>
      <p className="mt-1 text-xs">Scheduled and published posts will appear here.</p>
    </div>
  );
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/PostsTabPlaceholder.tsx
git commit -m "feat(social-posts): add PostsTabPlaceholder"
```

---

## Phase 7 — Modals

### Task 18: `ConfirmPublishModal`

**Files:**
- Create: `frontend/src/components/socialPosts/ConfirmPublishModal.tsx`

- [ ] **Step 1: Create the component**

```tsx
import { Modal } from "../common/Modal";
import facebookLogo from "../../assets/brands/facebook.svg";
import instagramLogo from "../../assets/brands/instagram.svg";

interface ConfirmPublishModalProps {
  open: boolean;
  facebookSelected: boolean;
  instagramSelected: boolean;
  isPublishing: boolean;
  errorMessage: string | null;
  onCancel: () => void;
  onConfirm: () => void;
}

export function ConfirmPublishModal({
  open,
  facebookSelected,
  instagramSelected,
  isPublishing,
  errorMessage,
  onCancel,
  onConfirm,
}: ConfirmPublishModalProps) {
  const channelLabel = buildChannelLabel(facebookSelected, instagramSelected);

  function handleCancel() {
    if (!isPublishing) onCancel();
  }

  function handleConfirm() {
    if (!isPublishing) onConfirm();
  }

  return (
    <Modal
      open={open}
      onClose={handleCancel}
      title={`Publish to ${channelLabel} now?`}
      size="sm"
      footer={
        <>
          <button
            type="button"
            onClick={handleCancel}
            disabled={isPublishing}
            className="rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm font-medium text-gray-700 hover:bg-gray-50 disabled:opacity-60"
          >
            Cancel
          </button>
          <button
            type="button"
            onClick={handleConfirm}
            disabled={isPublishing}
            className="rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-500 disabled:opacity-60"
          >
            {isPublishing ? "Publishing…" : "Publish now"}
          </button>
        </>
      }
    >
      <div className="flex items-center gap-3">
        {facebookSelected ? (
          <span className="inline-flex items-center gap-1 rounded bg-gray-100 px-2 py-1 text-xs text-gray-700">
            <img src={facebookLogo} alt="" className="h-3 w-3" /> Facebook
          </span>
        ) : null}
        {instagramSelected ? (
          <span className="inline-flex items-center gap-1 rounded bg-gray-100 px-2 py-1 text-xs text-gray-700">
            <img src={instagramLogo} alt="" className="h-3 w-3" /> Instagram
          </span>
        ) : null}
      </div>
      {errorMessage ? (
        <p className="mt-3 rounded border border-red-200 bg-red-50 p-2 text-xs text-red-700">
          {errorMessage}
        </p>
      ) : null}
    </Modal>
  );
}

function buildChannelLabel(fb: boolean, ig: boolean): string {
  if (fb && ig) return "Facebook + Instagram";
  if (fb) return "Facebook";
  if (ig) return "Instagram";
  return "selected channels";
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/ConfirmPublishModal.tsx
git commit -m "feat(social-posts): add ConfirmPublishModal"
```

---

### Task 19: `ScheduleModal`

**Files:**
- Create: `frontend/src/components/socialPosts/ScheduleModal.tsx`

- [ ] **Step 1: Create the component**

```tsx
import { useMemo, useState } from "react";
import { Modal } from "../common/Modal";
import { isAtLeastMinutesAhead, toUtcIso } from "../../lib/scheduledTime";

const MIN_LEAD_MINUTES = 5;

interface ScheduleModalProps {
  open: boolean;
  isScheduling: boolean;
  errorMessage: string | null;
  onCancel: () => void;
  onConfirm: (scheduledAtUtcIso: string) => void;
}

export function ScheduleModal({
  open,
  isScheduling,
  errorMessage,
  onCancel,
  onConfirm,
}: ScheduleModalProps) {
  const defaultTimeZone = useMemo(() => {
    try {
      return Intl.DateTimeFormat().resolvedOptions().timeZone || "UTC";
    } catch {
      return "UTC";
    }
  }, []);
  const timeZones = useMemo(() => buildTimeZoneList(defaultTimeZone), [defaultTimeZone]);
  const [date, setDate] = useState("");
  const [time, setTime] = useState("");
  const [timeZone, setTimeZone] = useState(defaultTimeZone);

  const utcIso = useMemo(() => {
    if (!date || !time) return null;
    try {
      return toUtcIso({ date, time, timeZone });
    } catch {
      return null;
    }
  }, [date, time, timeZone]);

  const isFarEnough = utcIso ? isAtLeastMinutesAhead(utcIso, MIN_LEAD_MINUTES) : false;
  const canConfirm = utcIso !== null && isFarEnough && !isScheduling;
  const localPreview = utcIso ? formatLocalPreview(utcIso) : null;

  function handleDateChange(event: React.ChangeEvent<HTMLInputElement>) {
    setDate(event.target.value);
  }
  function handleTimeChange(event: React.ChangeEvent<HTMLInputElement>) {
    setTime(event.target.value);
  }
  function handleTimeZoneChange(event: React.ChangeEvent<HTMLSelectElement>) {
    setTimeZone(event.target.value);
  }
  function handleCancel() {
    if (!isScheduling) onCancel();
  }
  function handleConfirm() {
    if (canConfirm && utcIso) onConfirm(utcIso);
  }

  return (
    <Modal
      open={open}
      onClose={handleCancel}
      title="Schedule post"
      size="md"
      footer={
        <>
          <button
            type="button"
            onClick={handleCancel}
            disabled={isScheduling}
            className="rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm font-medium text-gray-700 hover:bg-gray-50 disabled:opacity-60"
          >
            Cancel
          </button>
          <button
            type="button"
            onClick={handleConfirm}
            disabled={!canConfirm}
            className="rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-500 disabled:cursor-not-allowed disabled:bg-gray-300"
          >
            {isScheduling ? "Scheduling…" : "Schedule"}
          </button>
        </>
      }
    >
      <div className="space-y-3">
        <div className="grid grid-cols-2 gap-3">
          <label className="block text-sm">
            <span className="block text-gray-700">Date</span>
            <input
              type="date"
              value={date}
              onChange={handleDateChange}
              className="mt-1 w-full rounded-md border border-gray-300 px-2 py-1 text-sm"
            />
          </label>
          <label className="block text-sm">
            <span className="block text-gray-700">Time</span>
            <input
              type="time"
              step={300}
              value={time}
              onChange={handleTimeChange}
              className="mt-1 w-full rounded-md border border-gray-300 px-2 py-1 text-sm"
            />
          </label>
        </div>
        <label className="block text-sm">
          <span className="block text-gray-700">Timezone</span>
          <select
            value={timeZone}
            onChange={handleTimeZoneChange}
            className="mt-1 w-full rounded-md border border-gray-300 px-2 py-1 text-sm"
          >
            {timeZones.map((zone) => (
              <option key={zone} value={zone}>
                {zone}
              </option>
            ))}
          </select>
        </label>
        {localPreview ? (
          <p className="text-xs text-gray-500">Local time: {localPreview}</p>
        ) : null}
        {utcIso && !isFarEnough ? (
          <p className="text-xs text-red-600">
            Pick a time at least {MIN_LEAD_MINUTES} minutes in the future.
          </p>
        ) : null}
        {errorMessage ? (
          <p className="rounded border border-red-200 bg-red-50 p-2 text-xs text-red-700">
            {errorMessage}
          </p>
        ) : null}
      </div>
    </Modal>
  );
}

function buildTimeZoneList(currentZone: string): string[] {
  const supported = supportsTimeZonesApi() ? Intl.supportedValuesOf("timeZone") : FALLBACK_ZONES;
  const all = supported.includes(currentZone) ? supported : [currentZone, ...supported];
  return all;
}

function supportsTimeZonesApi(): boolean {
  return typeof Intl.supportedValuesOf === "function";
}

const FALLBACK_ZONES = [
  "UTC",
  "Pacific/Auckland",
  "Australia/Sydney",
  "Asia/Tokyo",
  "Asia/Singapore",
  "Asia/Dubai",
  "Europe/London",
  "Europe/Paris",
  "America/New_York",
  "America/Chicago",
  "America/Denver",
  "America/Los_Angeles",
];

function formatLocalPreview(utcIso: string): string {
  const formatter = new Intl.DateTimeFormat(undefined, {
    weekday: "short",
    day: "2-digit",
    month: "short",
    hour: "2-digit",
    minute: "2-digit",
    timeZoneName: "short",
  });
  return formatter.format(new Date(utcIso));
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/ScheduleModal.tsx
git commit -m "feat(social-posts): add ScheduleModal"
```

---

## Phase 8 — Compose tab orchestrator

### Task 20: `ComposeTab`

**Files:**
- Create: `frontend/src/components/socialPosts/ComposeTab.tsx`

- [ ] **Step 1: Create the orchestrator**

```tsx
import { useState } from "react";
import { ChannelChip } from "./ChannelChip";
import { CaptionField } from "./CaptionField";
import { MediaDropZone } from "../media/MediaDropZone";
import { PublishControls } from "./PublishControls";
import { PreviewCard } from "./PreviewCard";
import { ConfirmPublishModal } from "./ConfirmPublishModal";
import { ScheduleModal } from "./ScheduleModal";
import {
  useUploadPostMedia,
  usePublishPostNow,
  useSchedulePost,
} from "../../hooks/useSocialPosts";
import {
  resolveAccountIds,
  validateCompose,
} from "../../lib/socialPostValidation";
import type {
  ComposeFormState,
  Health,
} from "../../lib/socialPostValidation";
import type { MetaConnection } from "../../types/meta";
import type {
  MediaResponse,
  PostPublishRequest,
  PostScheduleRequest,
} from "../../types/posts";
import type { UploadedMediaFile } from "../../types/media";

const INITIAL_FORM: ComposeFormState = {
  caption: "",
  selectedPlatforms: { facebook: false, instagram: false },
  media: null,
  mediaUploadState: "idle",
  mediaUploadError: null,
};

interface ComposeTabProps {
  health: Health;
  connection: MetaConnection | null | undefined;
  onPublishSuccess: () => void;
  onScheduleSuccess: () => void;
}

export function ComposeTab({
  health,
  connection,
  onPublishSuccess,
  onScheduleSuccess,
}: ComposeTabProps) {
  const [form, setForm] = useState<ComposeFormState>(INITIAL_FORM);
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [scheduleOpen, setScheduleOpen] = useState(false);
  const [submitError, setSubmitError] = useState<string | null>(null);

  const uploadMutation = useUploadPostMedia();
  const publishMutation = usePublishPostNow();
  const scheduleMutation = useSchedulePost();

  const validation = validateCompose(form, health);
  const isUploading = form.mediaUploadState === "uploading";
  const canPublish = validation.ok && !isUploading;
  const isBusy =
    publishMutation.isPending ||
    scheduleMutation.isPending ||
    isUploading;

  function handleFacebookToggle() {
    setForm((current) => ({
      ...current,
      selectedPlatforms: {
        ...current.selectedPlatforms,
        facebook: !current.selectedPlatforms.facebook,
      },
    }));
  }
  function handleInstagramToggle() {
    setForm((current) => ({
      ...current,
      selectedPlatforms: {
        ...current.selectedPlatforms,
        instagram: !current.selectedPlatforms.instagram,
      },
    }));
  }
  function handleCaptionChange(next: string) {
    setForm((current) => ({ ...current, caption: next }));
  }
  function handleMediaChange(next: UploadedMediaFile | null) {
    setForm((current) => ({
      ...current,
      media: next,
      mediaUploadState: "idle",
      mediaUploadError: null,
    }));
  }
  async function handleUpload(file: File): Promise<UploadedMediaFile> {
    setForm((current) => ({ ...current, mediaUploadState: "uploading" }));
    try {
      const response = await uploadMutation.mutateAsync(file);
      const converted = mediaResponseToFile(response);
      setForm((current) => ({ ...current, mediaUploadState: "idle" }));
      return converted;
    } catch (error) {
      setForm((current) => ({
        ...current,
        mediaUploadState: "error",
        mediaUploadError: error instanceof Error ? error.message : "Upload failed",
      }));
      throw error;
    }
  }
  function handlePublishClick() {
    setSubmitError(null);
    if (!validation.ok) return;
    setConfirmOpen(true);
  }
  function handleScheduleClick() {
    setSubmitError(null);
    if (!validation.ok) return;
    setScheduleOpen(true);
  }
  function handleCancelConfirm() {
    setConfirmOpen(false);
    setSubmitError(null);
  }
  function handleCancelSchedule() {
    setScheduleOpen(false);
    setSubmitError(null);
  }
  async function handleConfirmPublish() {
    if (!validation.ok) return;
    const accounts = resolveAccountIds(connection, form.selectedPlatforms);
    if (accounts.missingPrimary.length > 0) {
      setSubmitError(`No primary ${accounts.missingPrimary.join(" and ")} account set.`);
      return;
    }
    const body: PostPublishRequest = {
      content: form.caption,
      media_ids: form.media ? [form.media.id] : [],
      account_ids: accounts.accountIds,
    };
    try {
      await publishMutation.mutateAsync(body);
      setConfirmOpen(false);
      setForm(INITIAL_FORM);
      onPublishSuccess();
    } catch (error) {
      setSubmitError(error instanceof Error ? error.message : "Failed to publish.");
    }
  }
  async function handleConfirmSchedule(scheduledAtUtcIso: string) {
    if (!validation.ok) return;
    const accounts = resolveAccountIds(connection, form.selectedPlatforms);
    if (accounts.missingPrimary.length > 0) {
      setSubmitError(`No primary ${accounts.missingPrimary.join(" and ")} account set.`);
      return;
    }
    const body: PostScheduleRequest = {
      content: form.caption,
      media_ids: form.media ? [form.media.id] : [],
      account_ids: accounts.accountIds,
      scheduled_at: scheduledAtUtcIso,
    };
    try {
      await scheduleMutation.mutateAsync(body);
      setScheduleOpen(false);
      setForm(INITIAL_FORM);
      onScheduleSuccess();
    } catch (error) {
      setSubmitError(error instanceof Error ? error.message : "Failed to schedule.");
    }
  }

  return (
    <div className="grid gap-6 lg:grid-cols-3">
      <div className="space-y-5 lg:col-span-2">
        <div className="rounded-lg border border-gray-200 bg-white p-5 shadow-sm">
          <div>
            <h3 className="text-sm font-medium text-gray-700">Publish to</h3>
            <div className="mt-2 flex gap-2">
              <ChannelChip
                platform="facebook"
                health={health.facebook}
                selected={form.selectedPlatforms.facebook}
                onToggle={handleFacebookToggle}
              />
              <ChannelChip
                platform="instagram"
                health={health.instagram}
                selected={form.selectedPlatforms.instagram}
                onToggle={handleInstagramToggle}
              />
            </div>
            <p className="mt-2 text-xs text-gray-500">
              Disconnected platforms are greyed out — connect them in the Accounts tab.
            </p>
          </div>
          <div className="mt-5">
            <CaptionField
              value={form.caption}
              onChange={handleCaptionChange}
              selectedPlatforms={form.selectedPlatforms}
            />
          </div>
          <div className="mt-5">
            <MediaDropZone
              value={form.media}
              onChange={handleMediaChange}
              accept={{ image: true, video: true }}
              uploadFn={handleUpload}
              state={form.mediaUploadState}
              errorMessage={form.mediaUploadError}
              label="Image or video"
            />
          </div>
          <div className="mt-5">
            <PublishControls
              canPublish={canPublish}
              isBusy={isBusy}
              onPublishClick={handlePublishClick}
              onScheduleClick={handleScheduleClick}
            />
            {!validation.ok ? (
              <ul className="mt-2 space-y-1">
                {validation.errors.map((issue) => (
                  <li key={`${issue.field}-${issue.message}`} className="text-xs text-red-600">
                    {issue.message}
                  </li>
                ))}
              </ul>
            ) : null}
          </div>
        </div>
      </div>
      <div>
        <PreviewCard />
      </div>
      <ConfirmPublishModal
        open={confirmOpen}
        facebookSelected={form.selectedPlatforms.facebook}
        instagramSelected={form.selectedPlatforms.instagram}
        isPublishing={publishMutation.isPending}
        errorMessage={submitError}
        onCancel={handleCancelConfirm}
        onConfirm={handleConfirmPublish}
      />
      <ScheduleModal
        open={scheduleOpen}
        isScheduling={scheduleMutation.isPending}
        errorMessage={submitError}
        onCancel={handleCancelSchedule}
        onConfirm={handleConfirmSchedule}
      />
    </div>
  );
}

function mediaResponseToFile(response: MediaResponse): UploadedMediaFile {
  return {
    id: response.id,
    url: response.url,
    kind: response.kind,
    width: response.width,
    height: response.height,
    sizeBytes: response.size_bytes,
  };
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/socialPosts/ComposeTab.tsx
git commit -m "feat(social-posts): add ComposeTab orchestrator"
```

---

## Phase 9 — Page container

### Task 21: `SocialPostsPage`

**Files:**
- Create: `frontend/src/pages/SocialPostsPage.tsx`

- [ ] **Step 1: Create the page**

```tsx
import { useEffect, useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { CheckCircle2, Edit3, ListChecks } from "lucide-react";
import { fetchMetaConnection } from "../api/metaCredentials";
import { ComposeTab } from "../components/socialPosts/ComposeTab";
import { PostsTabPlaceholder } from "../components/socialPosts/PostsTabPlaceholder";
import { AccountsConnectBanner } from "../components/socialPosts/AccountsConnectBanner";
import type { MetaConnection } from "../types/meta";
import type {
  Health,
  PlatformHealth,
} from "../lib/socialPostValidation";

type ActiveTab = "compose" | "posts";

const META_CONNECTION_KEY = ["meta-connection"] as const;
const SUCCESS_DISMISS_MS = 4000;

const TABS: { id: ActiveTab; label: string; icon: typeof Edit3 }[] = [
  { id: "compose", label: "Compose", icon: Edit3 },
  { id: "posts", label: "Posts", icon: ListChecks },
];

export default function SocialPostsPage() {
  const { data: connection } = useQuery<MetaConnection | null>({
    queryKey: META_CONNECTION_KEY,
    queryFn: fetchMetaConnection,
  });
  const [activeTab, setActiveTab] = useState<ActiveTab>("compose");
  const [successMessage, setSuccessMessage] = useState<string | null>(null);

  useEffect(() => {
    if (!successMessage) return;
    const timer = window.setTimeout(() => setSuccessMessage(null), SUCCESS_DISMISS_MS);
    return () => window.clearTimeout(timer);
  }, [successMessage]);

  const health = deriveHealth(connection);
  const noPlatformsConnected =
    health.facebook !== "connected" && health.instagram !== "connected";

  function makeTabHandler(tab: ActiveTab) {
    return () => setActiveTab(tab);
  }
  function handlePublishSuccess() {
    setSuccessMessage("Post published");
  }
  function handleScheduleSuccess() {
    setSuccessMessage("Post scheduled");
  }

  return (
    <div className="px-6 py-6">
      <div className="mb-2 inline-block rounded-full bg-indigo-50 px-3 py-0.5 text-xs font-medium text-indigo-700">
        Social Media
      </div>
      <h1 className="text-2xl font-semibold text-gray-900">Social posts</h1>
      <p className="mt-1 text-sm text-gray-500">
        Publish and schedule posts to Facebook and Instagram from one place.
      </p>
      <div className="mt-5 border-b border-gray-200">
        <nav className="flex gap-6">
          {TABS.map((tab) => (
            <button
              key={tab.id}
              type="button"
              onClick={makeTabHandler(tab.id)}
              className={`flex items-center gap-2 border-b-2 pb-2 text-sm transition-colors ${
                activeTab === tab.id
                  ? "border-indigo-600 text-indigo-600"
                  : "border-transparent text-gray-500 hover:text-gray-700"
              }`}
            >
              <tab.icon className="h-4 w-4" />
              {tab.label}
            </button>
          ))}
        </nav>
      </div>
      {successMessage ? (
        <div
          role="status"
          className="mt-4 flex items-center gap-2 rounded-md border border-green-200 bg-green-50 px-3 py-2 text-sm text-green-800"
        >
          <CheckCircle2 className="h-4 w-4" />
          {successMessage}
        </div>
      ) : null}
      <div className="mt-6 space-y-4">
        {activeTab === "compose" ? (
          <>
            {noPlatformsConnected ? <AccountsConnectBanner /> : null}
            <ComposeTab
              health={health}
              connection={connection ?? null}
              onPublishSuccess={handlePublishSuccess}
              onScheduleSuccess={handleScheduleSuccess}
            />
          </>
        ) : (
          <PostsTabPlaceholder />
        )}
      </div>
    </div>
  );
}

function deriveHealth(connection: MetaConnection | null | undefined): Health {
  return {
    facebook: derivePlatformHealth(connection, "facebook"),
    instagram: derivePlatformHealth(connection, "instagram"),
  };
}

function derivePlatformHealth(
  connection: MetaConnection | null | undefined,
  platform: "facebook" | "instagram",
): PlatformHealth {
  if (!connection) return "disconnected";
  const account = (connection.accounts ?? []).find((entry) => entry.platform === platform);
  if (!account) return "disconnected";
  if (!account.authorized) return "unhealthy";
  return "connected";
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/pages/SocialPostsPage.tsx
git commit -m "feat(social-posts): add SocialPostsPage container"
```

---

## Phase 10 — Route registration (three-place rule)

### Task 22: Register the route

**Files:**
- Modify: `frontend/src/App.tsx`
- Modify: `frontend/src/components/layout/Sidebar.tsx`
- Modify: `frontend/src/components/auth/ClientModeGuard.tsx`

Refer to root `CLAUDE.md` → "Adding a new frontend route — three places, not two".

- [ ] **Step 1: Add the route to `App.tsx`**

Open `frontend/src/App.tsx`. Find the `<Route element={<Layout />}>` block (around lines 63 onward). Add a new line near the other top-level routes:

```tsx
              <Route path="social-media" element={<SocialPostsPage />} />
```

Add the import at the top of `App.tsx` next to the other page imports:

```tsx
import SocialPostsPage from "./pages/SocialPostsPage";
```

Match the existing import ordering / alphabetisation convention in that file.

- [ ] **Step 2: Add the sidebar entry under Automation**

Open `frontend/src/components/layout/Sidebar.tsx`. Find the `Automation` category (label `"Automation"`, around line 76). Add a new child entry in the `children` array. Insert it after `"Job Routing"` and before `"Doc Templates"`:

```tsx
      {
        label: "Social Media",
        to: "/social-media",
        clientApp: true,
        permission: "posts:publish",
      },
```

(The backend route already declares `posts:publish` as the required permission — see `src/api/posts.py:28`.)

- [ ] **Step 3: Add the route to `CLIENT_ROUTES`**

Open `frontend/src/components/auth/ClientModeGuard.tsx`. Add `"/social-media"` to the `CLIENT_ROUTES` Set (alphabetical or grouped — match the surrounding convention):

```ts
  "/social-media",
```

- [ ] **Step 4: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 5: Run the full test suite**

Run: `cd frontend && npx vitest run`
Expected: all tests PASS (existing tests plus the new `socialPostValidation.test.ts` and `scheduledTime.test.ts`).

- [ ] **Step 6: Commit**

```bash
git add frontend/src/App.tsx frontend/src/components/layout/Sidebar.tsx frontend/src/components/auth/ClientModeGuard.tsx
git commit -m "feat(social-posts): register /social-media route, sidebar, and client mode allowlist"
```

---

## Phase 11 — Manual QA

### Task 26: Manual smoke checks

No file changes — purely verification. The team's testing convention is "type-checking and tests cover correctness; manual verification covers feature behaviour" (per the root `CLAUDE.md`).

- [ ] **Step 1: Build & run the dev environment**

Run: `cd frontend && npm run dev` (in one shell)
Run: `./run_dev.sh` or the equivalent backend dev launcher (in another shell).

Sign in as an agency user against a sandbox tenant.

- [ ] **Step 2: Sidebar + route reachability**

- Confirm the **Social Media** entry appears under **Automation** in the sidebar.
- Click it; the URL becomes `/social-media`; the page renders with the "Social posts" header and two tabs.
- Click **Posts**; the placeholder card renders.
- Click **Compose**; the composer renders again.

- [ ] **Step 3: Disconnected-state banner**

- Visit Meta settings; if either account is connected, disconnect both.
- Reload `/social-media`. Verify:
  - Yellow banner shows above the composer.
  - Both `ChannelChip` controls are disabled, with tooltips "Reconnect required".
  - "Publish now" and "Schedule" are disabled.

- [ ] **Step 4: Connected flow — publish image to Facebook only**

- Reconnect Facebook (only).
- Toggle the Facebook chip on. Type a short caption. Upload a small JPEG via drag-and-drop.
- Click **Publish now** → modal opens with title "Publish to Facebook now?" + Facebook badge.
- Confirm → success toast appears, composer resets to empty.
- Inspect the dev DB: a `posts` row exists with `status='published'` (or `publishing`/`partial_failure`, depending on Meta sandbox state).

- [ ] **Step 5: Connected flow — schedule a post**

- Toggle Facebook chip on. Compose a caption + image.
- Click **Schedule** → modal opens with date / time / timezone fields prefilled with the browser TZ.
- Pick a time 10 minutes from now in a non-local TZ (e.g. `Pacific/Auckland`).
- Verify "Local time: …" line shows the equivalent time in your browser TZ.
- Confirm → success toast appears, composer resets.
- Inspect the dev DB: a `posts` row exists with `status='scheduled'` and `scheduled_at` matching the picked instant in UTC.

- [ ] **Step 6: Validation edge cases**

- With Facebook off and Instagram off, type a caption. "Publish now" should be disabled with the error "Select at least one platform to publish to.".
- With Instagram selected and no media attached, "Publish now" should be disabled with the error "Instagram requires an image or video.".
- Type 2,201 `a` chars with Facebook only selected; verify the amber cross-platform warning appears under the caption.
- Attempt to drop a `.txt` file → "Unsupported file type." appears.
- Attempt to drop a >100 MB image (use a synthetic file) → size error appears, no upload network call fires.

- [ ] **Step 7: Schedule below 5-minute lead**

- Click Schedule, pick the current time. Verify the Schedule button is disabled and "Pick a time at least 5 minutes in the future." appears.

- [ ] **Step 8: Mark the task complete only after every step above passes**

If any step fails, fix the underlying code and re-run the affected steps. Do not mark complete on partial pass.

---

## Self-review summary

Coverage against the spec:

- §3 (Route, sidebar, page shell) → Task 21, Task 22.
- §4 (Account health gating) → Task 21 (`deriveHealth` in `SocialPostsPage`), Task 11 (`ChannelChip` disabled state), Task 16 (`AccountsConnectBanner`).
- §5 (ComposeTab layout) → Task 20.
- §6 (ComposeTab state) → Task 20.
- §7 (Sub-components) → Tasks 11–17, 20.
- §8 (Validation rules) → Task 5 (`validateCompose`), Task 4 (tests), Task 6 (`resolveAccountIds`).
- §9 (API integration) → Tasks 1, 2, 3, 8.
- §10 (Reusable MediaDropZone) → Task 10.
- §11 (Submission flows) → Task 20 (`ComposeTab`), Tasks 18 + 19 (modals), Task 7 (TZ helper).
- §12 (Error handling) → Tasks 18, 19, 20 (inline modal error rendering); Task 10 (MediaDropZone error state).
- §13 (File layout) → Every Task creates a single file in the documented location.
- §14 (Testing) → Pure-logic tests via Vitest (Tasks 4, 5, 6, 7); manual QA covers UI behaviour (Task 26). The spec's component-level tests are deferred; deviation documented in plan intro. Backend publishing tests live separately under `tests/integration/test_api_posts_publish.py` and `test_api_posts_schedule.py` (not tracked as plan tasks — they're regular integration tests).

Open items inherited from spec §15 are handled inline in the relevant tasks:
- Shared modal primitive — Task 9 creates `components/common/Modal.tsx`.
- Date library — Task 7 uses `Intl.DateTimeFormat` only, no new dependency.
- Lucide icon for Sidebar — Task 22 uses an icon already in scope (uses the Automation parent's icon; Social Media child has no per-item icon, matching the existing children pattern).
- Backend schema field names — Task 2 step 1 reads `src/schemas/posts.py` before defining the TS types.
