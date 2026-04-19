# Toonwide Web App — Product Specification

## 1. Overview and Vision

Toonwide is a web application for reading webtoon comics using a **horizontal segment layout**. Traditional webtoon readers force users to scroll vertically through extremely long strips. Toonwide takes a different approach: image segments (pre-processed by the companion Electron desktop app) are packed into columns that fit the viewport height and laid out horizontally. This lets readers — especially those with ultrawide monitors — see significantly more content in their viewport at once.

### How It Works (End to End)

1. An admin uses the **Electron desktop app** to process raw chapter images: stitching them into a single vertical strip, removing blank gaps, and splitting the result into manageable **WebP lossless** segments (~20-30% smaller than PNG with no quality loss).
2. The admin uploads those segments to the **Toonwide web app** through the Admin Dashboard.
3. Segments are stored in **Cloudflare R2** (S3-compatible object storage).
4. Public users visit the site, browse comics, and read chapters in the **horizontal reader**.

---

## 2. Tech Stack

The project is scaffolded in `wt-web-app/` using the `create-mugnavo` template.

| Layer | Technology |
|-------|------------|
| UI Framework | React 19 + React Compiler |
| Meta-framework | TanStack Start + TanStack Router + TanStack Query |
| Styling | Tailwind CSS + shadcn/ui + Base UI |
| Build | Vite 8 + Nitro v3 |
| Database | PostgreSQL (via Docker Compose) |
| ORM | Drizzle ORM (snake_case, PostgreSQL dialect) |
| Auth | Better Auth (Email OTP + Google OAuth), Resend (email delivery) |
| Image Storage | Cloudflare R2 (production), MinIO (local development) |
| Linting/Formatting | Oxlint + Oxfmt |
| Package Manager | pnpm |

**Additional packages to install** (not included in the template):

- `@aws-sdk/client-s3` — S3-compatible client for R2/MinIO
- `@aws-sdk/s3-request-presigner` — presigned URL generation for direct uploads
- `resend` — email delivery for OTP codes
- `@tanstack/react-table` — admin data tables
- `react-hook-form` + `@hookform/resolvers` — form state management for admin forms
- `tsx` (dev dependency) — running the seed script standalone

---

## 3. File Organization Conventions

New domain code follows the same structure as the existing `src/lib/auth/` directory. Each domain gets its own folder under `src/lib/` with a consistent set of files.

```
src/
├── lib/
│   ├── auth/                          # Existing auth domain
│   │   ├── auth.ts                    # Better Auth config (server-only)
│   │   ├── auth-client.ts            # Client-side auth
│   │   ├── functions.ts              # Server functions ($getUser)
│   │   ├── queries.ts                # TanStack Query options (authQueryOptions)
│   │   ├── hooks.ts                  # React hooks (useAuth, useAuthSuspense)
│   │   └── middleware.ts             # authMiddleware, adminMiddleware
│   ├── comic/                         # Comic domain
│   │   ├── functions.ts              # $getComics, $createComic, $updateComic, etc.
│   │   ├── queries.ts                # comicQueryOptions, comicsQueryOptions, etc.
│   │   └── schemas.ts                # Zod schemas for comic server function inputs
│   ├── chapter/                       # Chapter domain
│   │   ├── functions.ts
│   │   ├── queries.ts
│   │   └── schemas.ts
│   ├── segment/                       # Segment domain
│   │   ├── functions.ts
│   │   ├── queries.ts
│   │   └── schemas.ts
│   ├── tag/                           # Tag domain
│   │   ├── functions.ts
│   │   └── queries.ts
│   ├── bookmark/                      # Bookmark domain
│   │   ├── functions.ts
│   │   └── queries.ts
│   ├── reading-history/               # Reading history domain
│   │   ├── functions.ts
│   │   └── queries.ts
│   ├── storage/                       # S3/R2 client
│   │   └── s3.ts                     # Shared S3Client instance + helper functions
│   ├── db/
│   │   ├── index.ts                  # Drizzle client (existing)
│   │   ├── seed.ts                   # Seed script
│   │   └── schema/
│   │       ├── index.ts              # Re-exports all schemas
│   │       ├── auth.schema.ts        # Existing: user, session, account, verification
│   │       ├── comic.schema.ts       # comic, tag, comicTag tables + relations
│   │       ├── chapter.schema.ts     # chapter table + relations
│   │       ├── segment.schema.ts     # segment table + relations
│   │       └── user-feature.schema.ts # bookmark, readingHistory tables + relations
│   └── utils.ts                      # Shared utilities (cn, slugify, etc.)
```

### Rules

- **One schema file per domain** in `src/lib/db/schema/`, re-exported from `index.ts`.
- **Server functions** in `src/lib/{domain}/functions.ts`, defined with `createServerFn` and prefixed with `$`.
- **TanStack Query options** in `src/lib/{domain}/queries.ts`, wrapping server functions for use in route loaders and components.
- **Zod validation schemas** in `src/lib/{domain}/schemas.ts`, imported by `functions.ts` for `.validator()` calls.
- **S3 client** as a singleton in `src/lib/storage/s3.ts`, configured from environment variables.
- **Middleware** (`adminMiddleware`) lives alongside `authMiddleware` in `src/lib/auth/middleware.ts`.

---

## 4. Data Model

All new tables use Drizzle ORM and live in `src/lib/db/schema/`. The existing auth tables (`user`, `session`, `account`, `verification`) remain unchanged except for the addition of a `role` column on `user`.

### 4.1 Entity Relationship Diagram

```mermaid
erDiagram
    user ||--o{ bookmark : "has"
    user ||--o{ readingHistory : "has"
    comic ||--o{ chapter : "contains"
    comic ||--o{ bookmark : "bookmarked by"
    comic ||--o{ comicTag : "tagged with"
    comic ||--o{ comicDailyView : "tracked by"
    tag ||--o{ comicTag : "applied to"
    chapter ||--o{ segment : "contains"
    chapter ||--o{ readingHistory : "read in"

    user {
        text id PK
        text name
        text email UK
        text role "admin | user"
        text image
        timestamp createdAt
        timestamp updatedAt
    }

    comic {
        serial id PK
        text slug UK
        text originalName
        text localizedName
        text description
        text coverImageUrl
        text author
        text artist
        text status "ongoing | completed | hiatus"
        integer viewCount
        timestamp publishedAt "nullable - null is draft"
        timestamp createdAt
        timestamp updatedAt
    }

    tag {
        serial id PK
        text name UK
        text slug UK
    }

    comicTag {
        serial id PK
        integer comicId FK
        integer tagId FK
    }

    chapter {
        serial id PK
        integer comicId FK
        integer number
        text name
        text slug
        text thumbnailUrl
        integer viewCount
        timestamp publishedAt "nullable - null is draft"
        timestamp createdAt
        timestamp updatedAt
    }

    segment {
        serial id PK
        integer chapterId FK
        integer sortOrder
        text imageUrl
        integer width
        integer height
        integer fileSize "bytes, nullable"
        text topGapColor "hex, nullable"
        text bottomGapColor "hex, nullable"
        text topEdgeStrip "data URI, nullable"
        boolean topEdgeStripIsLight "nullable"
        text bottomEdgeStrip "data URI, nullable"
        boolean bottomEdgeStripIsLight "nullable"
        timestamp createdAt
    }

    bookmark {
        serial id PK
        text userId FK
        integer comicId FK
        timestamp createdAt
    }

    readingHistory {
        serial id PK
        text userId FK
        integer comicId FK
        integer chapterId FK
        timestamp lastReadAt
    }

    comicDailyView {
        serial id PK
        integer comicId FK
        date date
        integer viewCount
    }
```

### 4.2 Table Details

#### `user` (extend existing)

Add a `role` column to the existing Better Auth user table.

| Column | Type | Notes |
|--------|------|-------|
| role | `text` | `"admin"` or `"user"`, default `"user"` |

#### `comic`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| slug | `text` | Unique, URL-friendly identifier |
| originalName | `text` | Original title (e.g. in Korean/Japanese) |
| localizedName | `text` | Translated/localized title |
| description | `text` | Synopsis, nullable |
| coverImageUrl | `text` | R2 URL for cover image, nullable. Not set on initial creation -- admin uploads the cover via the edit page after the comic record exists (since the R2 object key requires the `comicId`). |
| author | `text` | Writer name |
| artist | `text` | Artist name (may be same as author) |
| status | `text` | One of: `ongoing`, `completed`, `hiatus` |
| viewCount | `integer` | Total views, default 0 |
| publishedAt | `timestamp` | Nullable. `null` = draft (not visible to public). When set, the comic is live as of that timestamp. A future timestamp means scheduled. |
| createdAt | `timestamp` | Auto-set on insert |
| updatedAt | `timestamp` | Auto-set on insert and update. In Drizzle, use `.$onUpdate(() => new Date())` since Drizzle does not auto-update timestamps. |

#### `tag`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| name | `text` | Display name (e.g. "Action"), unique |
| slug | `text` | URL-friendly version, unique |

#### `comicTag` (join table)

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| comicId | `integer` | FK → `comic.id`, cascade delete |
| tagId | `integer` | FK → `tag.id`, cascade delete |

Unique constraint on `(comicId, tagId)`.

#### `chapter`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| comicId | `integer` | FK → `comic.id`, cascade delete |
| number | `integer` | Chapter number within the comic |
| name | `text` | Chapter title, nullable |
| slug | `text` | URL-friendly identifier |
| thumbnailUrl | `text` | R2 URL for chapter thumbnail, nullable |
| viewCount | `integer` | Default 0 |
| publishedAt | `timestamp` | Nullable. `null` = draft. When set, the chapter is live as of that timestamp. Drives the "Recently Updated" queries on public pages (not `createdAt`). |
| createdAt | `timestamp` | Auto-set on insert |
| updatedAt | `timestamp` | Auto-set on insert and update |

Unique constraint on `(comicId, number)`.
Unique constraint on `(comicId, slug)`.

#### `segment`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| chapterId | `integer` | FK → `chapter.id`, cascade delete |
| sortOrder | `integer` | Display order within the chapter (0-indexed) |
| imageUrl | `text` | R2 URL for the segment image |
| width | `integer` | Natural image width in pixels |
| height | `integer` | Natural image height in pixels |
| fileSize | `integer` | File size in bytes, nullable. Captured during upload for admin storage stats. |
| topGapColor | `text` | Hex color of removed gap above this segment, nullable. Set from `metadata.json` during upload. |
| bottomGapColor | `text` | Hex color of removed gap below this segment, nullable. Set from `metadata.json` during upload. |
| topEdgeStrip | `text` | `data:image/png;base64,...` URI of the 1px-tall top content edge, nullable. Set from `metadata.json` during upload. Used for gradient gap rendering at manual split boundaries. |
| bottomEdgeStrip | `text` | `data:image/png;base64,...` URI of the 1px-tall bottom content edge, nullable. Set from `metadata.json` during upload. Used for gradient gap rendering at manual split boundaries. |
| topEdgeStripIsLight | `boolean` | Whether the top edge strip's average luminance is light (> 128), nullable. Used by the reader's column bottom gap to choose a brightness-aware fill color. |
| bottomEdgeStripIsLight | `boolean` | Whether the bottom edge strip's average luminance is light (> 128), nullable. Used by the reader's column bottom gap to choose a brightness-aware fill color. |
| topGapColorOverride | `text` | Admin-set hex color override for the top gap, nullable. When non-null, takes precedence over `topGapColor` in the reader (merged via `COALESCE` at query time). |
| bottomGapColorOverride | `text` | Admin-set hex color override for the bottom gap, nullable. When non-null, takes precedence over `bottomGapColor` in the reader (merged via `COALESCE` at query time). |
| isExcluded | `boolean` | Admin toggle: hides from public reader without deleting. Default `false`. Excluded segments still count toward storage stats. |
| gaplessBelow | `boolean` | Admin toggle: 0px gap between this segment and the next in the reader. Images touch directly. Default `false`. |
| createdAt | `timestamp` | Auto-set on insert |

#### `bookmark`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| userId | `text` | FK → `user.id`, cascade delete |
| comicId | `integer` | FK → `comic.id`, cascade delete |
| createdAt | `timestamp` | Auto-set on insert |

Unique constraint on `(userId, comicId)`.

#### `readingHistory`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| userId | `text` | FK → `user.id`, cascade delete |
| comicId | `integer` | FK → `comic.id`, cascade delete |
| chapterId | `integer` | FK → `chapter.id`, cascade delete |
| lastReadAt | `timestamp` | Updated each time the user opens this chapter |

Unique constraint on `(userId, chapterId)`.

#### `comicDailyView`

| Column | Type | Notes |
|--------|------|-------|
| id | `serial` | Primary key |
| comicId | `integer` | FK → `comic.id`, cascade delete |
| date | `date` | Calendar day |
| viewCount | `integer` | Views accumulated for this comic on this day, default 0 |

Unique constraint on `(comicId, date)`.

Each chapter read upserts a row for the parent comic and today's date, incrementing `viewCount` by 1. Used for time-range popularity sorting on the browse page.

### 4.3 Drizzle Relations

Each schema file must also export Drizzle `relations()` definitions, following the existing pattern in `auth.schema.ts` (which exports `userRelations`, `sessionRelations`, `accountRelations`). Relations are required for efficient joins in query functions (e.g. fetching a comic with its tags, or a chapter with its segment count).

Expected relation exports per schema file:

- `comic.schema.ts`: `comicRelations`, `tagRelations`, `comicTagRelations`, `comicDailyViewRelations`
- `chapter.schema.ts`: `chapterRelations`
- `segment.schema.ts`: `segmentRelations`
- `user-feature.schema.ts`: `bookmarkRelations`, `readingHistoryRelations`

### 4.4 Database Indexes

PostgreSQL does not automatically create indexes on foreign key columns. Define explicit indexes on FK and frequently-filtered columns, following the existing `auth.schema.ts` pattern (e.g. `index("session_userId_idx").on(table.userId)`).

**Indexes covered by unique constraints** (automatically indexed, no action needed):
- `comic.slug` (unique)
- `tag.name`, `tag.slug` (unique)
- `(comicId, tagId)` on `comicTag` (composite unique)
- `(comicId, number)` on `chapter` (composite unique)
- `(comicId, slug)` on `chapter` (composite unique)
- `(userId, comicId)` on `bookmark` (composite unique)
- `(userId, chapterId)` on `readingHistory` (composite unique)
- `(comicId, date)` on `comicDailyView` (composite unique)

**Indexes to define explicitly:**

| Table | Index | Reason |
|-------|-------|--------|
| `comic` | `publishedAt` | Filtered on every public query |
| `chapter` | `comicId` | Lookup chapters by comic |
| `chapter` | `publishedAt` | Filtered on every public query |
| `segment` | `chapterId` | Lookup segments by chapter |
| `comicTag` | `comicId` | Lookup tags for a comic |
| `comicTag` | `tagId` | Lookup comics for a tag |
| `bookmark` | `userId` | Lookup bookmarks for a user |
| `bookmark` | `comicId` | Check if a comic is bookmarked |
| `readingHistory` | `userId` | Lookup history for a user |
| `readingHistory` | `comicId` | Lookup history for a comic |
| `readingHistory` | `chapterId` | Check if a chapter has been read |
| `comicDailyView` | `comicId` | Lookup daily views for a comic |
| `comicDailyView` | `date` | Time-range filtering for popularity sort |

---

## 5. Image Storage

### 5.1 Production — Cloudflare R2

- S3-compatible API; use the AWS SDK (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`).
- Single bucket (e.g. `toonwide-images`).
- Object key convention: `comics/{comicId}/cover.{ext}`, `comics/{comicId}/chapters/{chapterId}/thumb.{ext}`, `comics/{comicId}/chapters/{chapterId}/segments/{sortOrder}.{ext}`.
- Public read access enabled on the bucket (or served through Cloudflare CDN).

### 5.2 Development — MinIO

Add a MinIO service to `docker-compose.yml` alongside PostgreSQL:

```yaml
minio:
  image: minio/minio
  ports:
    - 9000:9000
    - 9001:9001
  volumes:
    - minio_data:/data
  environment:
    - MINIO_ROOT_USER=minioadmin
    - MINIO_ROOT_PASSWORD=minioadmin
  command: server /data --console-address ":9001"
```

MinIO exposes the same S3 API, so the application code uses a single S3 client configured via environment variables.

**Bucket initialization**: The seed script creates the `toonwide-images` bucket (if it doesn't exist) with a public-read policy and configures CORS to allow presigned PUT uploads from the dev server origin (`http://localhost:2110`). Without CORS, the browser blocks direct uploads to MinIO.

### 5.3 Environment Variables

```
R2_ENDPOINT=http://localhost:9000          # MinIO locally, R2 endpoint in prod
R2_ACCESS_KEY_ID=minioadmin
R2_SECRET_ACCESS_KEY=minioadmin
R2_BUCKET=toonwide-images
R2_PUBLIC_URL=http://localhost:9000/toonwide-images  # Public URL prefix for serving images
```

### 5.4 Upload Flow

**Segment uploads** (multiple files at once):

1. Admin selects files in the browser (drag-and-drop or file picker).
2. Client sorts files by filename using **numeric-aware sorting** (`localeCompare` with `{ numeric: true }`) so `2.png` comes before `10.png`.
3. Client reads each file's natural dimensions and file size via the `Image` and `File` APIs.
4. Client calls a server function to request **presigned PUT URLs** for each file (short-lived, ~5 minutes, scoped to the correct object key).
5. Client uploads files directly to R2/MinIO using the presigned URLs, **limiting concurrency** to 3-5 parallel uploads to avoid overwhelming the connection. Shows per-file progress and overall "X of N uploaded" status.
6. On upload completion, the client calls a server function to batch-create segment metadata (URL, width, height, fileSize, sortOrder) in the database.

**Cover image and thumbnail uploads** (single file):

Cover images and chapter thumbnails are uploaded via the **edit** form, not the create form. This is because the R2 object key requires the entity ID (e.g., `comics/{comicId}/cover.{ext}`), which only exists after the record is created. The flow is:

1. Admin creates the comic/chapter (without image) via the create form.
2. Admin is redirected to the edit form.
3. Admin selects an image file; the client shows a local preview via `URL.createObjectURL`.
4. Client requests a presigned PUT URL using the known entity ID.
5. Client uploads directly to R2/MinIO.
6. Client calls the update server function to set the image URL on the record.

---

## 6. User Roles and Authentication

### 6.1 Roles

| Role | Access |
|------|--------|
| `admin` | Full access: Admin Dashboard + all public pages |
| `user` | Public pages + bookmarks, reading history |
| Anonymous | Public pages only (browse and read) |

### 6.2 Implementation

- Add a `role` column via Better Auth's `user.additionalFields` config in `src/lib/auth/auth.ts` (not by manually editing `auth.schema.ts`, which is auto-generated). Set `type: "string"`, `defaultValue: "user"`, `input: false` (prevents users from setting their own role via signup). After updating the config, run `pnpm auth:generate` to regenerate the schema with the new column.
- Create an `adminMiddleware` in `src/lib/auth/middleware.ts` that **chains on the existing `authMiddleware`** (inheriting the authenticated user context), then checks `context.user.role === "admin"` and returns 403 if not.
- Admin accounts are created manually (e.g. via a DB seed script or by updating the role column directly) — there is no public admin registration.

### 6.3 Auth Methods

- **Email OTP** (passwordless): Uses Better Auth's `emailOTP` plugin. The user enters their email, receives a 6-digit code, and enters the code to sign in. This handles both new users (account created automatically) and returning users in a single flow -- no separate signup page needed.
- **Google OAuth**: Single social sign-in provider via Better Auth's `socialProviders.google` config.

### 6.4 Email Delivery

The `emailOTP` plugin requires a `sendVerificationOTP` callback in `src/lib/auth/auth.ts` to deliver OTP codes:

- **Production**: Send via Resend (`RESEND_API_KEY` env var).
- **Development**: Log the OTP code to the server console (`console.log`) so developers can copy it without needing a real email service. Optionally also send via Resend if the API key is configured.

### 6.5 Session Management

- **Session cookie caching** (5-minute TTL, already configured in the template).
- Hooks: `useAuth()` and `useAuthSuspense()` from `src/lib/auth/hooks.ts`.

---

## 7. Admin Dashboard

Protected under the `/_admin` route group. Only users with `role === "admin"` can access these pages. The layout includes a sidebar with navigation between admin sections.

### 7.1 Dashboard Overview (`/_admin/`)

- Total comics count (with draft/published breakdown)
- Total chapters count (with draft/published breakdown)
- Total segments count
- Recently updated comics (last 10)

### 7.2 Comic Management (`/_admin/comics`)

#### Comic List

- Paginated table of all comics (including drafts).
- **Server-side search**: search input filters the paginated table by comic name or author (via an optional `search` param on `$getAdminComics`).
- Columns: cover thumbnail, localized name, original name, author, status, publish state (Draft / Published / Scheduled), view count, chapter count, last updated.
- Actions: Edit, Delete (with confirmation dialog).
- "Create Comic" button.

#### Create / Edit Comic

Form fields:

| Field | Input Type | Validation |
|-------|-----------|------------|
| Original Name | Text input | Required |
| Localized Name | Text input | Required |
| Slug | Text input | Auto-generated from localized name, editable. Unique. |
| Description | Textarea | Optional |
| Cover Image | File upload + crop (single image) | Optional. Not available on the create form (R2 key requires `comicId`). Admin uploads via the edit page after creation. After file selection, a crop dialog opens automatically with a fixed 2:3 portrait ratio. The cropped WebP result is what gets uploaded to R2. |
| Author | Text input | Required |
| Artist | Text input | Required |
| Status | Select (ongoing / completed / hiatus) | Required, default "ongoing" |
| Tags | Multi-select / tag input | Optional, select from existing or create new |
| Publish | Radio group: Save as Draft / Publish Now / Schedule | Default "Save as Draft". Schedule shows a date-time picker. |

### 7.3 Chapter Management (`/_admin/comics/$comicId/chapters`)

#### Chapter List

- Table of chapters for the selected comic, ordered by chapter number (including drafts).
- **Drag-and-drop reordering** updates chapter numbers to match the new order.
- **Inline chapter name editing**: click the name to edit in place.
- The **full list is loaded** (no pagination) so reordering applies to all chapters at once.
- A **0-segment warning** indicator highlights chapters that have no segment content yet.
- Columns: thumbnail, number, name, publish state (Draft / Published / Scheduled), segment count, view count, published date.
- Actions: Edit, Delete, Manage Segments.

#### Create / Edit Chapter

| Field | Input Type | Validation |
|-------|-----------|------------|
| Number | Number input | Required, unique within comic |
| Name | Text input | Optional (e.g. "The Beginning") |
| Slug | Text input | Auto-generated from number + name, editable |
| Thumbnail | Crop from segment (see below) | Optional. Only on edit page (requires segments). |
| Publish | Radio group: Save as Draft / Publish Now / Schedule | Default "Save as Draft". Schedule shows a date-time picker. |

#### Chapter Thumbnail (Crop from Segment)

The edit page includes a **Thumbnail** section above the form fields. Thumbnails are created by cropping a segment image rather than uploading a separate file. The crop uses a fixed **2:3 portrait** aspect ratio (matching comic covers).

**Workflow:**
1. Admin clicks "Set Thumbnail from Segment" (or "Change" if one already exists).
2. A dialog opens with **Step 1 — Segment Picker**: a scrollable grid of the chapter's segments. Segments too small for a valid crop are dimmed and unselectable.
3. Admin clicks a segment to advance to **Step 2 — Crop**: the segment is displayed in a `react-easy-crop` component with drag, zoom, and a fixed 2:3 crop window.
4. A zoom slider and real-time pixel readout (e.g., "Crop: 520 × 780 px") are displayed below the crop area.
5. A minimum crop width of **480px** is enforced (save button disabled below this). Segments whose dimensions make it impossible to reach 480px are blocked at the picker step with a "Too small" label.
6. On save, the cropped region is drawn to an offscreen canvas, exported as **WebP** (quality 1.0), uploaded to R2 via presigned URL (`comics/{comicId}/chapters/{chapterId}/thumb.webp`), and `chapter.thumbnailUrl` is updated.
7. Output is capped at **600px wide** (600×900). Crops between 480–600px use natural size (no scaling).

**Removal:** A "Remove" button sets `thumbnailUrl` to null. The R2 object is cleaned up on the next thumbnail upload (prefix delete in the presigned URL function).

**Note:** Thumbnails persist after segment re-uploads (replace-all semantics only clean the `segments/` prefix, not `thumb.webp`). The admin can manually re-crop if new segments differ.

#### Batch Import (`/_admin/comics/$comicId/chapters/batch-import`)

- Select a folder (via `webkitdirectory`) containing sub-folders of processed chapter outputs from the Desktop App.
- Each sub-folder with image files becomes an importable chapter.
- Folder name convention: `[Toonwide] 001-chapter-name` — the `[Toonwide]` prefix and leading numeric prefix are stripped to extract the chapter name. Chapter numbers are **not** parsed from folder names; they are auto-assigned sequentially starting after the last existing chapter for the comic, avoiding collisions.
- Preview table shows detected chapters with editable numbers, names, slugs, and publish state. The page prefetches existing chapters to determine the starting number.
- Imports all selected chapters in batch (creates chapter records, then uploads segments per chapter sequentially).
- Progress UI shows per-chapter status during import.

### 7.4 Segment Manager (`/_admin/comics/$comicId/chapters/$chapterId/segments`)

- Displays all segments for the chapter in a grid (ordered by `sortOrder`).
- **Upload**: Drag-and-drop zone or file picker accepting multiple images plus an optional `metadata.json` from the desktop split output. Files are sorted by filename (numeric-aware) before upload. Gap colors are auto-extracted from `metadata.json` and attached to each segment on create.
- **Upload semantics (replace-all)**: Every upload **replaces all** existing segments for the chapter. There is no append behavior — new files become the full segment set for the chapter. Re-uploading is equivalent to replacing the chapter’s images in one step.
- **Delete**: Per-segment delete is not available. **Delete All** bulk-removes every segment for the chapter (R2 + database). Clearing can also be done implicitly by uploading a new set (replace-all).
- **Preview**: Thumbnail of each segment with its dimensions and file size displayed.
- **Set as Thumbnail**: Each segment card shows a "Thumbnail" crop button on hover (segments too small for a 2:3 crop at 480px are excluded). Clicking opens the `ImageCropDialog` directly, bypassing the segment picker step from the chapter edit page. The cropped WebP is uploaded to R2 and `chapter.thumbnailUrl` is updated.
- **Thumbnail indicator**: The top of the page shows the chapter's current thumbnail (if set) with a "Remove" option, or a prompt to set one by clicking a segment.
- **Storage summary**: Total file size for the chapter shown at the top of the segment list (sum of all segment `fileSize` values).
- On upload, the client reads each image's natural dimensions and file size before requesting presigned URLs, so width/height/fileSize metadata is stored alongside the URL.
- **Gap color overrides**: Each segment card always shows a gap color section (even when no colors are present) so admins can identify which boundaries use edge strip crossfade. A palette icon opens a popover to set hex color overrides for the top and bottom gap colors. Overrides are stored in separate DB columns (`topGapColorOverride`, `bottomGapColorOverride`) so the originals from `metadata.json` are preserved. Setting an override to null (clearing the input) restores the original. Both sides of a boundary must have effective gap colors to replace a crossfade — the UI includes a hint explaining this. Override swatches are shown with a dashed primary border to distinguish them from originals.
- **Segment exclusion**: Each card has an eye/eye-off toggle (top-left of thumbnail) to hide the segment from the public reader without deleting it. Excluded cards are dimmed (`opacity-40`) with a "Hidden" badge. The toggle calls `$toggleSegmentExclusion`. Excluded segments still count toward storage stats. Re-uploading (replace-all) resets all exclusions since rows are deleted and recreated.
- **Gapless below**: Each card has a chain-link icon toggle in the gap colors row. When active, the reader renders 0px gap between this segment and the next (images touch directly). Gapless pairs get an accent left border (`border-l-primary`) on both the initiator card and the next card, making chained segments visually obvious at a glance. The toggle calls `$toggleSegmentGaplessBelow`. Gapless applies in rendered order (after exclusions) — if the next segment is excluded, gapless applies to the segment after that.

---

## 8. Public Pages

All public pages apply a **visibility filter**: only comics and chapters where `publishedAt` is non-null and `<= now()` are shown. Drafts and scheduled-but-not-yet-live entries are invisible to non-admin users.

### 8.1 Home Page (`/`)

- **Popular carousel** (full viewport width, no header): Multi-slide auto-rotating carousel of top published comics by `viewCount`. Each slide is a **square (1:1)** card with the cover image as background, a dark gradient overlay, and the title/tags/latest chapter overlaid. Auto-rotates every 5 seconds, pauses on hover, loops infinitely when 2+ items. Responsive: 2 visible on mobile, 3-4 on tablet, 5-6 on desktop/ultrawide. Hidden when no popular comics exist.
- **Recently Updated**: Responsive grid of comics with new published chapters, showing comic card (cover, localized name, tags, latest chapter). Ordered by most recent chapter `publishedAt`. Hidden when empty.
- **Ongoing**: Responsive grid of published comics with status "ongoing", sorted by recently updated. Hidden when empty.
- **Completed**: Responsive grid of published comics with status "completed", sorted by recently updated. Hidden when empty.
- Grid sections use `max-w-7xl` container (consistent with the navbar) with responsive columns (2-6).
- Each comic card links to the Comic Detail page.

### 8.2 Browse Page (`/browse`)

- **Search bar**: Full-text search across `localizedName`, `originalName`, `author`, `artist`. Search uses PostgreSQL `ILIKE` with `%query%` wildcards, combining fields with `OR` for any-field matching. No external search service needed at this scale.
- **Filter sidebar/toolbar**:
  - Tags (multi-select checkboxes)
  - Status (ongoing / completed / hiatus)
  - Sort by: Recently Updated, Popular (with time range: This Week / This Month / This Year / All Time), A-Z
- **Results grid**: Paginated comic cards (cover, localized name, tags, published chapter count). Only published comics are included.
- Filters are reflected in URL search params for shareability.

### 8.3 Comic Detail Page (`/comic/$comicSlug`)

- **Header**: Cover image, localized name, original name, author, artist, status badge, view count, tags.
- **Description**: Full synopsis.
- **Bookmark button**: Visible if logged in; toggles bookmark state.
- **Chapter list**: Only published chapters are shown. Ordered by number (ascending or descending, toggleable). Each row shows chapter number, name, and published date. Clicking a chapter navigates to the reader. If the user is logged in, chapters they have read are visually marked.

### 8.4 Chapter Reader (`/comic/$comicSlug/chapter/$chapterSlug`)

This is the core feature. Detailed spec in Section 9.

---

## 9. Chapter Reader — Core Feature

The reader implements the horizontal column-packing layout from the prototype (`webtoon-prototype/docs/WEB_VIEWER_APP.md`), adapted for server-loaded images.

### 9.1 Layout Algorithm

```
Input:  segments[] (each with imageUrl, width, height)
        boardWidth  (measured content-box width of the reader area)
        boardHeight (measured content-box height of the reader area)

Step 1 — Adaptive width selection (computeOptimalMaxWidth):
  minWidth   = max(ABSOLUTE_MIN_WIDTH, boardWidth * VIEWPORT_MIN_WIDTH_RATIO)
  upperBound = min(segment natural widths)  — cap ensures uniform scaling

  If upperBound <= minWidth: use upperBound directly

  Otherwise, grid search from minWidth to upperBound (1px steps):
    For each candidate maxWidth, run column packing and score by
    average column bottom gap.  Pick the candidate with the lowest
    average gap.  Tiebreaker (within 0.5px): prefer closest to
    DEFAULT_MAX_WIDTH (360).

Step 2 — Column packing (computeColumns, greedy sequential):
  For each segment:
    scale = min(1, maxWidth / naturalWidth, boardHeight / naturalHeight)
    scaledWidth  = naturalWidth  * scale
    scaledHeight = naturalHeight * scale

    if currentColumnHeight + scaledHeight > boardHeight AND column not empty:
      finalize column, start new one
    add segment to current column

  finalize last column
```

**Why adaptive width**: With a fixed `maxWidth`, the scaled segment heights are locked to one scale, and whether segments pack tightly into columns or leave large bottom gaps is pure coincidence. By searching over maxWidth values, the algorithm finds the scale where segments fill columns most efficiently — minimizing the visible dead space at the bottom of each column.

**Constants and settings**:

| Value | Source | Default | Purpose |
|---|---|---|---|
| `DEFAULT_MAX_WIDTH` | Internal constant | `360` | Fallback width and tiebreaker anchor |
| `minColumnWidthFloor` | Reader settings (Section 9.10) | `300` | Hard floor — columns never narrower than this |
| `viewportWidthRatio` | Reader settings (Section 9.10) | `0.1` | Minimum width as fraction of board width |

**Upper bound = min(natural widths)**: Capping maxWidth at the narrowest segment's natural width guarantees every segment is uniformly scaled by `maxWidth / width` (no segment hits the scale=1 cap from width). This produces consistent column widths.

Segments within a column are **always contiguous in chapter order** — the algorithm never reorders segments across columns.

### 9.2 Layout Container

- **Desktop (>= 768px)**: A collapsible fixed sidebar on the left edge (`w-6` collapsed). The board fills the full viewport height with a left margin (`ml-6`) to clear the sidebar. `overflow-x: auto; overflow-y: hidden` — horizontal scroll is the primary reading axis.
- **Mobile (< 768px)**: A horizontal top bar above the board. The board fills the remaining viewport height below the bar. `overflow-x: hidden; overflow-y: auto` — vertical scroll.
- Each column is a vertical flex container; segments are stacked top-to-bottom with seams between images (16px–96px depending on gap type — see Section 9.7).
- The board dimensions are measured via `ResizeObserver` and recalculated on window resize. Layout mode (desktop vs mobile) is determined by `window.matchMedia`, decoupled from the board measurement to avoid feedback with the sidebar width.

### 9.3 Navigation

| Action | Trigger |
|--------|---------|
| Jump to chapter | Chapter select dropdown in sidebar (desktop), prev/next buttons in top bar (mobile) and sidebar |
| Next chapter | Click the end-of-chapter box after the last column (desktop), or prev/next buttons in sidebar/top bar |
| Previous chapter | Prev/next buttons in sidebar (desktop) or top bar (mobile) |
| Scroll by one column | Left/Right Arrow keys (smooth scroll to next/previous column edge, including the end-of-chapter box) |
| Free-form horizontal scroll | Mouse wheel, trackpad, horizontal scroll gesture |

**Arrow key column-step scrolling**: Each Left/Right Arrow key press scrolls the board by exactly one column (smooth animation via `scrollTo`). Pre-computed column edge positions (`columnEdges` array) serve as snap targets for content columns only (the end-of-chapter box is not a snap target). The next/previous edge relative to the current `scrollLeft` is found with a 1px tolerance to handle floating-point column widths. Arrow keys are purely scroll-only — they do **not** navigate between chapters. At the scroll boundaries they simply stop.

**End-of-chapter box**: A 300px-wide full-height box always renders after the last column. When a next chapter exists, it shows a clickable "Next Chapter" link with the chapter number and name. When this is the latest chapter, it shows a non-clickable "You're all caught up!" indicator. The box is not a `columnEdges` snap target — it is reachable via natural scrolling (wheel, trackpad) past the last content column. Since the box is never blurred (per-column blur only applies to content columns), it is always readable once visible.

**Trailing spacer**: A trailing spacer is always appended after the end-of-chapter box to ensure the last content column's `columnEdge` is reachable by ArrowRight. Its width is computed as `boardWidth - centerPadding - firstColWidth - boxWidth - N*gap`, where N is 3 when center-start is active or 2 otherwise — the exact minimum for the last column's edge to be reachable by `maxScroll`.

**Mouse wheel handling**: Since the reader scrolls horizontally but mouse wheels produce vertical `deltaY` events, the board must intercept `wheel` events and convert vertical scroll to horizontal: `boardRef.scrollLeft += event.deltaY`. Without this, standard mouse users cannot scroll the reader.

**Desktop sidebar** (collapsed by default):
- Toggle button expands the sidebar as an overlay showing: "Toonwide" logo (links to home page), comic title (links to comic detail), current chapter name/number, segment count, a chapter-select dropdown (displays chapter names, not slugs), and prev/next chapter buttons.
- Expanding/collapsing has no effect on board dimensions (sidebar is `fixed`, board `ml-6` is constant).

**Mobile top bar**:
- Displays: comic name, current chapter name/number, previous/next chapter buttons.
- If there is no next/previous chapter, the corresponding button is disabled.

### 9.4 Image Loading

- All segment metadata (including URLs and dimensions) is loaded in the route loader via TanStack Query (`ensureQueryData`).
- Since dimensions are known from the database, the layout can be computed **before** images finish loading, preventing layout shift.
- Images use `loading="lazy"` for off-screen segments (columns far to the right).
- Consider using `<img>` with explicit `width` and `height` attributes set to scaled dimensions for proper placeholder sizing.

### 9.5 Reading Progress

- When a logged-in user opens a chapter, upsert a `readingHistory` record with the current timestamp.
- This data is used on the Comic Detail page to highlight which chapters the user has already read.

### 9.6 View Counting

Chapter reads are the single source of truth for view counts. When a user opens a chapter, the `$incrementChapterView` server function (debounced — once per user session per chapter) performs three updates in a single operation:

1. Increment `chapter.viewCount` (cumulative counter for display on chapter rows)
2. Increment `comic.viewCount` on the parent comic (cumulative counter for display on cards and headers)
3. Upsert `comicDailyView(comicId, today)` incrementing `viewCount` by 1 (daily aggregate for time-range popularity sorting)

The comic detail page does **not** produce view counts — it only displays them. All views originate from chapter reads.

### 9.7 Gap rendering

Between consecutive segments in the same column, the reader renders a divider using one of five modes, checked in priority order. The `GapStrip` component selects the appropriate mode automatically. **Gapless is checked first** (admin-set 0px gap), then gap colors (admin overrides merged via `COALESCE` take precedence over edge strip crossfades):

| Priority | Condition | Rendering |
|---|---|---|
| 0 | **Gapless** (`prevSegment.gaplessBelow = true`) | No divider rendered (0px, images touch directly) |
| 1 | **Mismatched gap colors** (both non-null, different) | CSS `linear-gradient` blending the two colors (96px, `COLOR_GRADIENT_GAP_HEIGHT`) |
| 2 | **Matching gap colors** (both non-null, equal) | Solid `backgroundColor` (16px, `SOLID_GAP_HEIGHT`) |
| 3 | **Edge strips present** (manual split boundaries) | Edge strip crossfade with `plus-lighter` blending (96px, `EDGE_STRIP_GAP_HEIGHT`) |
| 4 | **Fallback** (one/both gap colors null, no edge strips) | Solid black fallback (16px, `SOLID_GAP_HEIGHT`) |

Gap heights are user-configurable via Reader Settings (Section 9.10): `solidGapHeight` (default 16px) for modes 2 and 4, `transitionGapHeight` (default 96px) for modes 1 and 3 (consolidated into one setting to reduce user confusion).

**Gap color override merging**: The segment table has two layers of gap color fields: originals (`topGapColor`, `bottomGapColor`) set from `metadata.json` during upload, and overrides (`topGapColorOverride`, `bottomGapColorOverride`) set by the admin via the Segment Manager. The `$getChapterBySlug` query merges them at query time using SQL `COALESCE(override, original)`, so the reader always sees a single effective value. This means admin overrides take precedence over originals, and clearing an override (setting to null) restores the original value. Since gap colors are checked before edge strips (priority 1-2 above 3), setting override colors on both sides of a crossfade boundary replaces the crossfade with color-based rendering.

**Edge strips**: When segments are manually split in the desktop app, interior boundaries get `null` gap colors. The processor extracts the 1px-tall content edge row as a PNG data URI (`topEdgeStrip` / `bottomEdgeStrip`), stored in `metadata.json` and the `segment` DB table. The reader tiles the strip vertically at native 1px height (`background-size: 100% 1px; background-repeat: repeat-y`) and fades opacity via CSS `mask-image`. When both adjacent segments have edge strips, they **crossfade** using `mix-blend-mode: plus-lighter` inside an `isolation: isolate` group — this makes alpha values add (0.5 + 0.5 = 1.0 at the midpoint) instead of compositing sequentially, eliminating dark bleed-through from the fallback background.

**`resolveGapColor`** is still used as the fallback background behind edge strip gradients and for the solid-color mode. Its resolution rules (prefer non-null, prefer distinctive over black/white) remain unchanged.

**Column bottom gap**: The vertical dead space below the last segment in a column uses a three-tier fill priority:

1. **`bottomGapColor` non-null** → solid gap color fill. A gap was removed below this segment, so we know the exact color. No gradient needed — the original image had a crisp content→gap boundary.
2. **`bottomEdgeStrip` present** (gap color null) → gradient fade spanning the full remaining column height. The content edge fades into a brightness-aware fill: `edgeStripFadeColor(bottomEdgeStripIsLight)` returns white for light strips, black for dark strips. This is for content boundaries where no gap was removed (e.g., end of chapter).
3. **Neither** → solid `resolveGapColor` fallback.

The first/last auto-split segments get edge strips extracted at chapter boundaries (where `gapColor` is null) so the gradient works for all chapters, not just manually-split ones.

Gaps **between columns** (the space between packed columns) use the user's `columnGap` (default 4px) and `columnGapColor` (default `#000000`) settings. On mobile, the board always uses `bg-black` regardless of the setting. The board element has the `dark` class forcing semantic colors to dark-theme values. Gap rendering applies only to in-column areas.

### 9.8 Responsive Behavior

The reader adapts its layout based on viewport width:

- **Above 768px** (desktop/ultrawide): Horizontal column-packing with adaptive width selection as described in Section 9.1. The board scrolls horizontally (`overflow-x: auto; overflow-y: hidden`).
- **Below 768px** (mobile/tablet): Falls back to a **single-column vertical layout**. All segments stack in one column at full container width. The adaptive `maxWidth` constraint is not used; segments scale to `min(1, containerWidth / naturalWidth)`. The board switches to vertical scroll (`overflow-x: hidden; overflow-y: auto`).

The layout mode is determined by `window.matchMedia('(min-width: 768px)')`, decoupled from the board's ResizeObserver measurement to avoid feedback with the fixed sidebar width.

### 9.9 Segment Preview

Clicking any segment image in the reader (both desktop and mobile) opens a full-viewport dark overlay showing the segment at its **original resolution**, scaled down only if it exceeds the viewport (32px padding on each side). The overlay uses `bg-black/90` and shows the segment number and natural dimensions in the top-left corner.

- **Open animation**: A hero/FLIP animation — the image flies from its source position in the reader to the centered dialog position over 300ms. The overlay UI (close button, segment info) fades in alongside the animation.
- **Close animation**: Reverse FLIP — the image flies back from center to its source position in the reader over 300ms. The overlay and UI elements fade out alongside the animation.

- **Close**: Click the empty area around the image, click the X button (top-right), or press Escape.
- **Zoom**: Native browser pinch-to-zoom and Ctrl+scroll-wheel zoom work inside the overlay for detailed inspection.
- **Broken images**: If a segment failed to load (showing the fallback placeholder), it is not clickable — the preview only opens for successfully loaded images.
- **Keyboard**: Arrow keys are disabled while the preview is open to prevent scrolling the board or navigating chapters behind the overlay.

### 9.10 Reader Settings

The reader provides inline settings controls in the expanded desktop sidebar as collapsible accordion sections (expanded by default). The expanded/collapsed state of each section is persisted in a separate `localStorage` key (`toonwide-settings-sections`) so it survives sidebar close/reopen. Settings values are stored in `localStorage` and apply immediately — no server involvement. The sidebar backdrop is transparent so users can see live layout changes while adjusting settings.

**Access**: Desktop sidebar only. Mobile users get whatever was configured on desktop via localStorage.

**Settings (8 total):**

| Setting | Default | Description |
|---|---|---|
| Solid gap height | 16px | Height of gaps between segments with matching colors |
| Transition gap height | 96px | Height of gaps for color transitions and split boundaries |
| Column spacing | 4px | Gap between columns |
| Column gap color | #000000 | Background color between columns (desktop only) |
| Min column width | 300px | Hard floor — columns are never narrower |
| Viewport width ratio | 0.1 | Min column width as fraction of screen (for ultrawides) |
| Start from center | on | First column centered on screen, upcoming columns blurred |
| Blur intensity | 8px | Strength of the anti-spoiler blur (visible when start-from-center is on) |

**Center-start mode with anti-spoiler blur**: When enabled, an invisible spacer aligns the first column's right edge with the viewport center, showing exactly one column of content. Unrevealed columns beyond the first are automatically blurred using per-column `filter: blur()`. As the user scrolls and a column's left edge crosses the board center, that column permanently unblurs with a 0.3s fade-in animation. Once a column is revealed, it **never re-blurs** — even if the user scrolls backward. This one-way reveal model is both simpler and more performant than the alternative of tracking scroll direction, and matches user intent (if you scrolled there, you already saw it). The blur intensity slider is only visible when start-from-center is on. The watermark (highest revealed segment `sortOrder`) survives column recomputation from resize or settings changes.

**Keyboard**: Arrow keys are disabled when focus is on settings sliders or switches.

**Reset**: A "Reset to Defaults" button restores all settings to their original values and re-expands all accordion sections.

---

## 10. Route Structure

TanStack Router file-based routing in `src/routes/`:

```
src/routes/
├── __root.tsx                                  # Root layout (theme, fonts, Toaster)
├── index.tsx                                   # Home page
├── browse.tsx                                  # Browse / search page
├── comic/
│   └── $comicSlug/
│       ├── index.tsx                           # Comic detail page
│       └── chapter/
│           └── $chapterSlug.tsx                # Chapter reader
├── _admin/
│   ├── route.tsx                               # Admin layout + role guard
│   ├── index.tsx                               # Dashboard overview
│   ├── tags.tsx                                # Tag management (list, create, delete)
│   └── comics/
│       ├── index.tsx                           # Comic list
│       ├── new.tsx                             # Create comic
│       └── $comicId/
│           ├── route.tsx                      # Layout: loads comic name into route context
│           ├── edit.tsx                        # Edit comic
│           └── chapters/
│               ├── index.tsx                   # Chapter list (DnD reordering, inline edit)
│               ├── new.tsx                     # Create chapter
│               ├── batch-import.tsx            # Batch chapter import
│               └── $chapterId/
│                   ├── route.tsx              # Layout: loads chapter info into route context
│                   ├── edit.tsx                # Edit chapter
│                   └── segments.tsx            # Segment upload & manager
├── _auth/
│   ├── route.tsx                               # Auth layout (existing)
│   └── app/
│       ├── route.tsx                           # Authenticated app layout (existing)
│       └── index.tsx                           # User dashboard: bookmarks
├── _guest/
│   ├── route.tsx                               # Guest layout (existing)
│   └── signin.tsx                              # Sign-in page (Email OTP + Google)
└── api/
    └── auth/
        └── $.ts                                # Better Auth API handler (existing)
```

### Route Guards

- `_admin/route.tsx`: Checks auth in `beforeLoad`. If no user, redirects to `/signin`. If user exists but `role !== "admin"`, redirects to `/`. Returns `{ user }` to child route contexts.
- `admin/comics/$comicId/route.tsx`: Layout route that fetches comic data in `beforeLoad` and provides `{ comic: { id, localizedName } }` to all children. Redirects to `/admin/comics` if the comic doesn't exist.
- `admin/comics/$comicId/chapters/$chapterId/route.tsx`: Layout route that fetches chapter data in `beforeLoad` and provides `{ chapter: { id, number, name } }` to children. Redirects to the chapters list if the chapter doesn't exist.
- `_auth/route.tsx`: Requires any authenticated user. Redirects to `/signin` if not logged in.
- `_guest/route.tsx`: Redirects authenticated users away from the sign-in page. Supports a `redirectUrl` search param for post-login navigation.
- Public routes (`/`, `/browse`, `/comic/*`): No auth required. Optionally load user data if a session exists (for bookmark state and reading history).

### Shell Opt-Out via `staticData`

The root layout in `__root.tsx` renders the public page shell (navbar + footer) by default. Routes that need a different layout set `staticData: { shell: false }` to opt out. The root layout checks `useMatches()` for this flag and conditionally renders the shell.

Routes that opt out of the public shell:
- `_admin/route.tsx` -- renders its own sidebar layout instead
- `_guest/route.tsx` -- sign-in page has no chrome (Spec Section 16.4)
- `comic/$comicSlug/chapter/$chapterSlug.tsx` -- reader has its own slim top bar (Spec Section 16.5)

---

## 11. Server Functions

All server functions use TanStack Start's `createServerFn` with the `$` prefix convention. Admin functions are protected by `adminMiddleware`.

**Input validation**: All server functions that accept input must use `.validator(zodSchema)` for server-side validation. Zod is already a project dependency. Define schemas in `src/lib/{domain}/schemas.ts` and import them into `functions.ts`. Example:

```typescript
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { createComicSchema } from "./schemas";

export const $createComic = createServerFn()
  .middleware([adminMiddleware])
  .validator(createComicSchema)
  .handler(async ({ data, context }) => {
    // data is typed and validated
  });
```

**Visibility rule**: All public query functions filter by `publishedAt IS NOT NULL AND publishedAt <= now()`. Admin query functions return all records regardless of publish state so admins can manage drafts and scheduled content.

**Pagination**: Functions that return lists use offset-based pagination with a standard contract:

- Input params: `{ page: number, pageSize: number }` (defaults: `page = 1`, `pageSize = 20`)
- Response shape: `{ items: T[], total: number, page: number, pageSize: number }`
- Paginated functions: `$getComics`, `$getAdminComics`, `$getChapters`, `$getBookmarks`
- Non-paginated list functions (small result sets, or full per-parent lists where pagination would break UI): `$getRecentlyUpdatedComics`, `$getPopularComics`, `$getTags`, `$getSegments`, `$getReadingHistory`, `$getAdminChapters` (all chapters for one comic — required for admin drag-and-drop reorder)

**Comic card data**: All functions that return comics for card rendering (`$getRecentlyUpdatedComics`, `$getPopularComics`, `$getComics`, `$getBookmarks`) must include the **latest published chapter number** per comic (for the "Ch. 45" label on cards). Use a subquery to get `MAX(chapter.number) WHERE publishedAt <= now()` per comic.

### 11.1 Comic Functions

| Function | Auth | Description |
|----------|------|-------------|
| `$getComics` | Public | List published comics with pagination, filtering, sorting. Filters by `publishedAt <= now()`. Supports sort options: Recently Updated (requires MAX(chapter.publishedAt) subquery), Popular (with time range via `comicDailyView` join), A-Z (ORDER BY localizedName). |
| `$getComicBySlug` | Public | Single published comic with tag and published chapter count info |
| `$getRecentlyUpdatedComics` | Public | Published comics ordered by most recent chapter `publishedAt`. Requires a subquery to get MAX(chapter.publishedAt) per comic. |
| `$getPopularComics` | Public | Published comics sorted by view count. Accepts an optional `timeRange` param (`week`, `month`, `year`, `all`). `all` uses `comic.viewCount`; others aggregate `comicDailyView` over the range. |
| `$getAdminComics` | Admin | List all comics (including drafts/scheduled) with pagination. Accepts optional `search` param for server-side filtering by comic name/author. |
| `$createComic` | Admin | Create a new comic (cover image is optional -- uploaded via edit page). `publishedAt` set per the form's publish control (null for draft, now for publish, future for scheduled). Checks slug uniqueness; appends numeric suffix on collision. |
| `$updateComic` | Admin | Update comic fields including `publishedAt` and `coverImageUrl` (cover upload happens here, not on create) |
| `$deleteComic` | Admin | Delete comic using a Drizzle transaction: cascade to chapters/segments in DB, and remove all associated R2 objects by prefix (`comics/{comicId}/`) |
| `$getDashboardStats` | Admin | Returns total comics/chapters/segments (with draft/published breakdown) and total storage size (sum of segment fileSize) |

### 11.2 Chapter Functions

| Function | Auth | Description |
|----------|------|-------------|
| `$getChapters` | Public | List published chapters for a comic. Filters by `publishedAt <= now()`. |
| `$getChapterBySlug` | Public | Single published chapter with segment metadata |
| `$getAdminChapters` | Admin | List all chapters for a comic (including drafts/scheduled) |
| `$batchCreateChapters` | Admin | Create multiple chapters in one operation (used by batch import after preview validation). |
| `$reorderChapters` | Admin | Persist chapter order after drag-and-drop on the chapter list (updates chapter numbers to match the new order). |
| `$createChapter` | Admin | Create a new chapter. `publishedAt` set per the form's publish control. |
| `$updateChapter` | Admin | Update chapter fields including `publishedAt` and `thumbnailUrl` (thumbnail set here after crop + presigned upload) |
| `$getChapterThumbnailPresignedUrl` | Admin | Generate a presigned PUT URL for uploading a cropped chapter thumbnail to R2. Deletes existing `thumb.*` objects first to prevent orphans on format change. |
| `$deleteChapter` | Admin | Delete chapter, cascade to segments in DB, and remove associated R2 objects (thumbnail, segments) |
| `$incrementChapterView` | Public | Debounced (once per session per chapter). Increments `chapter.viewCount`, increments parent `comic.viewCount`, and upserts `comicDailyView(comicId, today)` +1. |

### 11.3 Segment Functions

| Function | Auth | Description |
|----------|------|-------------|
| `$getSegments` | Public | List segments for a published chapter (ordered) |
| `$getPresignedUploadUrls` | Admin | Generate presigned PUT URLs for R2 |
| `$createSegments` | Admin | Batch-create segment metadata after upload (includes `fileSize`, optional `topGapColor` / `bottomGapColor` per segment when provided by the client, e.g. from `metadata.json`). Used as part of replace-all upload: existing segments for the chapter are cleared before new rows are written. |
| `$deleteAllSegments` | Admin | Bulk-deletes all segments for a chapter (R2 objects + DB rows). Validates chapter–comic ownership. |
| `$updateSegmentGapColorOverrides` | Admin | Updates the `topGapColorOverride` and `bottomGapColorOverride` on a single segment. Validates chapter–comic ownership. Setting to null clears the override, restoring the original gap color. |
| `$toggleSegmentExclusion` | Admin | Toggles `isExcluded` on a single segment. Excluded segments are hidden from the public reader but remain in the database. Validates chapter–comic ownership. |
| `$toggleSegmentGaplessBelow` | Admin | Toggles `gaplessBelow` on a single segment. When true, 0px gap between this segment and the next in the reader (images touch directly). Validates chapter–comic ownership. |

### 11.4 Tag Functions

| Function | Auth | Description |
|----------|------|-------------|
| `$getTags` | Public | List all tags. Includes **comic count per tag** (via LEFT JOIN comicTag + COUNT) for admin tag management display. |
| `$createTag` | Admin | Create a new tag |
| `$deleteTag` | Admin | Delete a tag |

### 11.5 User Feature Functions

| Function | Auth | Description |
|----------|------|-------------|
| `$toggleBookmark` | User | Add or remove a bookmark |
| `$getBookmarks` | User | List user's bookmarked comics |
| `$upsertReadingHistory` | User | Record that user opened a chapter |
| `$getReadingHistory` | User | Get user's reading history for a comic |

---

## 12. Environment Variables

Extend the existing `.env.example` with:

```
# Existing (remove GITHUB_* from template -- GitHub provider is not used)
VITE_BASE_URL=http://localhost:2110
DATABASE_URL="postgresql://postgres:password@localhost:5432/tanstarter"
BETTER_AUTH_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Email delivery (Resend) -- optional in dev (OTP codes are logged to console)
RESEND_API_KEY=

# Image Storage (R2 / MinIO)
R2_ENDPOINT=http://localhost:9000
R2_ACCESS_KEY_ID=minioadmin
R2_SECRET_ACCESS_KEY=minioadmin
R2_BUCKET=toonwide-images
R2_PUBLIC_URL=http://localhost:9000/toonwide-images
```

---

## 13. Seed Script

A seed script at `src/lib/db/seed.ts` populates the database with sample data for development. It should be runnable via `pnpm db:seed` (add a script entry in `package.json` using `tsx`: e.g. `"db:seed": "tsx src/lib/db/seed.ts"`).

### Behavior

- **MinIO bucket setup**: Creates the `toonwide-images` bucket (if it doesn't exist), sets a public-read bucket policy, and configures CORS rules to allow presigned PUT uploads from the dev server origin (`http://localhost:2110`). This must run before any image uploads will work.
- **Idempotent**: Truncates the seeded tables (`segment`, `chapter`, `comicTag`, `comicDailyView`, `comic`, `tag`, `bookmark`, `readingHistory`) and re-inserts. Does not touch auth tables except to upsert the admin user.
- **Admin user**: Creates or upserts a user with `role = "admin"` and email `admin@toonwide.com`. No password is stored (auth is passwordless). The admin signs in via Email OTP (code logged to server console in dev) or Google OAuth.
- **Tags**: Inserts a standard set: Action, Romance, Fantasy, Comedy, Slice of Life, Drama, Sci-fi, Horror.
- **Comics**: Creates 3 sample comics with all fields populated (names, descriptions, authors, cover image placeholder URLs, status, tags). All published (`publishedAt` set to past timestamps).
- **Chapters**: 3–5 chapters per comic, each with a name and number. All published.
- **Segments**: 5–10 segments per chapter with placeholder image URLs and realistic dimensions (e.g. 720x400, 720x600, 720x800). `fileSize` set to approximate values.
- **Daily views**: Seed `comicDailyView` with synthetic data spanning the last 30 days so the time-range popularity sort is testable. Vary the view counts across comics so sorting produces a visible difference.

This lets developers test all public pages and the reader without manually creating content through the admin dashboard.

---

## 14. Implementation Roadmap

Build the app in these phases. Each phase should result in working, testable functionality before moving to the next.

### Phase 1: Foundation

- Configure Better Auth: replace `emailAndPassword` with `emailOTP` plugin, remove GitHub social provider, keep Google. Wire up `sendVerificationOTP` callback (console log in dev, Resend in prod). Run `pnpm auth:generate` to confirm schema.
- Add `role` column to the existing user table in `auth.schema.ts`
- Create all Drizzle schema files (`comic.schema.ts`, `chapter.schema.ts`, `segment.schema.ts`, `user-feature.schema.ts`) and re-export from `index.ts`
- Run `pnpm db generate` and `pnpm db migrate` to apply migrations
- Add MinIO to `docker-compose.yml`
- Add R2/MinIO and Resend environment variables to `.env.example` and `.env`
- Create the S3 client module (`src/lib/storage/s3.ts`)
- Create the seed script and verify it populates data correctly
- Create `adminMiddleware` in `src/lib/auth/middleware.ts`
- Replace `login.tsx` + `signup.tsx` with a single `signin.tsx` page (Email OTP + Google button)

### Phase 2: Admin CRUD

- `_admin/route.tsx` layout with role guard and sidebar navigation
- Comic server functions, Zod schemas, and query options (`src/lib/comic/`)
- Comic admin pages: list, create, edit, delete
- Tag server functions and management UI
- Chapter server functions, schemas, query options (`src/lib/chapter/`)
- Chapter admin pages: list (DnD reorder, inline name edit), create, edit, delete, batch import
- Segment server functions, schemas, query options (`src/lib/segment/`)
- Segment manager page: upload via presigned URLs (with metadata.json gap color support; replace-all semantics), delete all, preview

### Phase 3: Public Pages

- Home page: recently updated comics, popular comics
- Browse page: search (ILIKE), filters (tags, status), sorting, pagination
- Comic detail page: metadata display, chapter list, view counting

### Phase 4: Chapter Reader

- Horizontal column-packing layout component
- `ResizeObserver`-based board measurement
- Keyboard navigation (arrow keys, chapter switching)
- Lazy image loading with pre-computed dimensions
- Responsive single-column fallback for mobile (below 768px)
- Chapter view counting

### Phase 5: User Features

- Wire up sign-in on public pages (sign-in prompt for bookmark/history actions)
- Bookmark toggle on comic detail page + bookmarks list
- Reading history tracking on chapter open
- Visual read/unread indicators on chapter lists

---

## 16. UI Design Guidelines

### 16.1 Global Design Principles

- **Dark theme by default**. Support light mode via the existing theme toggle (already in the template). All custom UI should look good in both themes, but design dark-first.
- **Cover-focused visual density**: large cover images dominate comic cards; text is secondary.
- **Typography**: use the existing DM Sans font (`@fontsource-variable/dm-sans`, already installed in the template).
- **Component library**: use shadcn/ui components as the base for all UI elements; extend with Tailwind for custom layouts.
- **Consistent styling**: rounded corners (`rounded-lg` / `rounded-xl`), subtle hover transitions (`transition-transform`, `hover:scale-[1.02]`), muted borders.

### 16.2 Page Shell

The public shell (navbar + footer) is rendered in `__root.tsx` using the `staticData` opt-out mechanism described in Section 10. Routes that set `staticData: { shell: false }` (admin, reader, sign-in) skip the shell and render their own layout.

**Top navbar** (sticky at top of viewport):

- Left: app logo/name (links to home).
- Center: search bar (`max-w-md`, flexes to fill available space). Submitting navigates to `/browse?q=<query>&page=1`; empty submit navigates to `/browse?page=1` (replaces the old Browse nav link). **Hidden on the browse page** (which has its own inline debounced search bar) — an empty flex spacer preserves layout spacing.
- Right: theme toggle, auth state. The navbar uses the `useAuth()` hook to get user data (no root-level `beforeLoad` prefetch needed). Anonymous users see a "Sign In" button. Logged-in users see a user avatar with a dropdown menu (Dashboard, Admin if admin, Sign Out).

**Footer** (at bottom of page content, not fixed):

- Minimal: app name, copyright.

**Routes without the public shell**:
- **Admin pages** (`_admin/*`): render their own sidebar layout.
- **Sign-in page** (`_guest/*`): centered card on minimal background, no chrome.
- **Chapter reader** (`comic/$comicSlug/chapter/$chapterSlug`): slim top bar, no navbar/footer.

### 16.3 Shared Components

#### Comic Card

Used on: home page, browse page, bookmarks dashboard.

- **Cover image** as the dominant element. Portrait aspect ratio (~2:3). Use `object-cover` to fill the card area.
- **Title** overlaid at the bottom of the cover with a dark-to-transparent gradient scrim, or displayed below the cover. White text on dark scrim for readability.
- **Tags** as small pill badges below the title (show 1-2 tags, "+N" for overflow).
- **Latest chapter** shown subtly (e.g. "Ch. 45" in a small muted label).
- **Hover state**: slight scale-up (`scale-[1.02]`) with an elevated shadow.
- **Responsive grid**: 2 columns on mobile, 3-4 on tablet, 5-6 on desktop/ultrawide.

#### Chapter Row

Used on: comic detail page chapter list.

- Horizontal row: chapter number (bold), chapter name, published date (muted, right-aligned).
- If the user is logged in: a read indicator (e.g. a subtle checkmark or dimmed opacity for read chapters). Unread chapters appear slightly brighter than read ones.
- Subtle divider between rows.
- Clickable -- the entire row navigates to the reader.

### 16.4 Public Pages

#### Home Page (`/`)

- **Popular carousel** (top, full viewport width, no header): Multi-slide Embla carousel with square (1:1) comic cards. Each card has the cover image as background with a dark gradient overlay and title/tags/chapter info overlaid. Auto-rotates (5s, `embla-carousel-autoplay`), pauses on hover, loops infinitely when 2+ items. Responsive slide count: `basis-1/2` mobile, `basis-1/3` tablet, `basis-1/4` to `basis-1/6` desktop/ultrawide. Navigation arrows appear on hover. Hidden if 0 comics; no arrows/autoplay if only 1.
- **Content grids** (contained in `max-w-7xl mx-auto px-4`, consistent with navbar): Three sections in order — "Recently Updated", "Ongoing", "Completed". Each section has a heading with "View all →" link (to browse with appropriate status filter) and a responsive `ComicCard` grid (`grid-cols-2` through `xl:grid-cols-6`). Sections are hidden when empty.
- Footer at the bottom.

#### Browse Page (`/browse`)

- **Search bar** at the top, prominent and full-width (or nearly).
- **Filter controls**: horizontal toolbar below the search bar with dropdown selects for tags, status, and sort order (including the time-range sub-options for popularity). On mobile, filters collapse into a "Filters" button that opens a slide-out drawer.
- **Results**: responsive grid of comic cards below the filters.
- **Pagination**: controls at the bottom of the grid (page numbers with prev/next buttons).
- **Empty state**: friendly message if no results match the filters.

#### Comic Detail Page (`/comic/$comicSlug`)

- **Top section** (two-column on desktop, stacked on mobile):
  - Left (~40% width): large cover image, full height.
  - Right (~60% width): localized name (large heading), original name (smaller, muted), author / artist, status badge (colored pill), view count, tags (pill badges).
  - Description (synopsis) below the metadata.
  - Bookmark button: prominent, in the metadata area. Heart icon or bookmark icon, toggles filled/outlined.
  - On mobile: cover full-width above metadata, stacked vertically.
- **Chapter list** below the top section:
  - Header row with "Chapters" title and a sort toggle button (ascending/descending by number).
  - List of chapter rows (shared Chapter Row component).
  - If many chapters, consider showing all with virtual scrolling or a "Show more" button.

#### Sign-in Page (`/signin`)

- Centered card on a minimal dark background. No shared navbar or footer (or minimal chrome).
- App logo at the top of the card.
- **Step 1**: email input + "Send Code" button.
- **Step 2** (after sending): 6-digit OTP code input (individual digit boxes or a single input) + "Verify" button. "Resend code" link.
- Divider: "or continue with".
- Google sign-in button (full-width, styled with Google branding).

### 16.5 Chapter Reader (`/comic/$comicSlug/chapter/$chapterSlug`)

- **Desktop** (above 768px):
  - **Collapsible left sidebar** (`fixed`, `w-6` collapsed rail / `w-80` expanded Sheet). Collapsed by default — shows only a small toggle button with tooltip ("Open menu") to maximize reading space. Expanded state uses shadcn Sheet (`side="left"`) with a transparent backdrop (click-to-close, board stays visible for live settings preview) and provides focus trap + Escape-to-close + slide animation. Expanded content: "Toonwide" logo text (links to home page), comic title (links to comic detail), chapter title, segment count, chapter-select dropdown (shows "Ch. N — Name" display labels; jump to any published chapter), and **inline accordion settings sections** for reader customization (see Section 9.10). Icon buttons have tooltips for accessibility.
  - **Board area**: fills the full viewport height (`h-svh`) with a left margin (`ml-6`) clearing the collapsed sidebar. Horizontal scroll, columns of segments as described in Section 9.
- **Mobile** (below 768px):
  - **Slim top bar**: back arrow (navigates to comic detail), comic title and chapter name/number (centered), prev/next chapter buttons (right). Semi-transparent background with backdrop blur.
  - **Board area**: fills the remaining viewport height below the top bar. Vertical scroll, single column, full-width segments.
- **Between segments**: in-column seams use dividers with user-configurable gap heights — see Section 9.7.
- **Reader settings**: Inline collapsible sections in the expanded sidebar. Controls gap heights, column spacing, column gap color, center-start mode, and anti-spoiler blur. Persisted in localStorage. See Section 9.10.
- **Segment preview**: Clicking any segment triggers a bidirectional hero animation — the image flies from its position in the reader to a full-viewport dark overlay, and flies back on close. `cursor-zoom-in` on hover signals clickability. See Section 9.9.
- No footer, no shared shell -- maximum reading space.

### 16.6 User Dashboard (`/_auth/app`)

- **Page title**: "My Bookmarks".
- **Content**: responsive grid of comic cards (same shared component as home/browse).
- **Empty state**: friendly illustration or icon, message like "No bookmarks yet. Browse comics to find something you love.", with a link/button to the browse page.
- No reading history UI here. Reading history only powers read/unread indicators on comic detail chapter lists.

### 16.7 Admin Pages

#### Admin Layout

- **Fixed left sidebar** (~240px width):
  - Top: app logo/name.
  - Nav items with icons: Dashboard (`LayoutDashboardIcon`), Comics (`BookOpenIcon`), Tags (`TagIcon`).
  - Bottom: signed-in user info (name, email) + Sign Out button.
  - On smaller screens: sidebar collapses to icon-only mode, expandable via hamburger.
- **Content area**: right of sidebar, with context-rich breadcrumbs at the top of each page. Breadcrumbs show the actual comic name and chapter number/name (loaded from layout route context), not just generic labels. Long names are truncated with `max-w-[20ch] truncate`. Chapters without names display as "Ch. N" only.

#### Dashboard Overview (`/_admin/`)

- **Stat cards** in a row (or 2x2 grid on mobile): Comics (showing published/draft counts), Chapters (published/draft), Segments (total count), Storage (total file size, human-readable).
- Below: "Recently Updated Comics" -- a simple list or compact table of the last 10 updated comics with links to edit.

#### Comic List (`/_admin/comics`)

- Toolbar: search input (filters table) + "Create Comic" button (primary action).
- shadcn **DataTable**: columns as specified in Section 7.2 (cover thumbnail, localized name, original name, author, status, publish state, view count, chapter count, last updated).
- Row actions via a dropdown menu ("..." button): Edit, Delete (with confirmation dialog).

#### Comic Form (create/edit)

- Single-column form layout with generous vertical spacing. Uses `react-hook-form` with Zod resolver for field-level validation and form state management.
- **Cover image** (edit page only): drag-and-drop zone or file picker. After file selection, a **crop dialog** opens automatically with a fixed 2:3 portrait ratio (same `ImageCropDialog` used for chapter thumbnails). The admin drags and zooms to frame the cover, then confirms. The cropped WebP blob replaces the raw file for upload. Not available on the create form because the R2 key requires `comicId`. After creating a comic, admin is redirected to the edit page to add the cover.
- Tags: multi-select combobox -- search existing tags, create new ones inline.
- Publish control: radio group at the bottom (Save as Draft / Publish Now / Schedule with date-time picker).
- Action buttons: Save + Cancel in a sticky footer bar at the bottom of the form.

#### Tag Management (`/_admin/tags`)

- "Add Tag" input + button at the top.
- List of existing tags, each showing: tag name, comic count, delete button (with confirmation).
- Simple, single-column layout.

#### Chapter List (`/_admin/comics/$comicId/chapters`)

- Breadcrumb: Comics > [Comic Name] > Chapters.
- Toolbar: "Create Chapter" and entry to **Batch Import** (Section 7.3).
- Full chapter list (no pagination): supports drag-and-drop reorder and inline name edit as in Section 7.3.
- shadcn **DataTable**: columns as specified in Section 7.3 (thumbnail, number, name, publish state, segment count, view count, published date).
- Row actions: Edit, Delete, Manage Segments.

#### Chapter Form (create/edit)

- Same layout pattern as comic form.
- Thumbnail: optional drag-and-drop zone with preview.
- Publish control: same radio group pattern.

#### Segment Manager (`/_admin/comics/$comicId/chapters/$chapterId/segments`)

- Breadcrumb: Comics > [Comic Name] > Ch. [Number]: [Name] > Segments.
- **Thumbnail indicator** at the top: shows the current chapter thumbnail (if set) with a "Remove" option, or a prompt to set one by clicking a segment's crop button.
- **Upload dropzone** below the thumbnail: large drag-and-drop area accepting multiple images.
- **Storage summary** bar below the dropzone: total file size for the chapter.
- **Grid** of segment thumbnails:
  - Each segment card: thumbnail image, dimensions label (e.g. "720x600"), file size label (e.g. "145 KB"), gap color swatches (when available), and a **"Thumbnail" crop button** on hover (opens `ImageCropDialog` for 2:3 crop). Segments too small for a valid crop are excluded from the crop action. No per-segment delete — use **Delete All** or replace via upload (Section 7.4).
  - Grid reflows responsively (2-4 columns depending on viewport).

### 16.8 Loading and Skeleton States

Every page that fetches data should show placeholder skeletons matching the shape of the expected content while loading. Use the shadcn `Skeleton` component.

| Page | Loading State |
|------|--------------|
| Home page | Carousel skeleton (full-width row of square shimmer cards) + three section skeletons (7 comic card placeholders each) |
| Browse page | Skeleton cards in the results grid; filters render immediately |
| Comic detail | Skeleton cover rectangle + text lines for metadata; skeleton rows for chapter list |
| Chapter reader | Dark board area with a centered spinner (segments load lazily after metadata) |
| User dashboard | Skeleton comic card grid |
| Admin lists | Skeleton table rows (5-6 rows matching column widths) |
| Admin forms | No skeleton needed (form fields render empty immediately) |

Route loaders using `ensureQueryData` will pre-fetch on the server, so full loading states are mainly visible on client-side navigations or slow connections. Still, always define them for a consistent experience.

### 16.9 shadcn/ui Component Mapping

Use these specific shadcn/ui components for common patterns. Install via `pnpm ui add <component>`.

| Pattern | Component |
|---------|-----------|
| Admin data tables | `Table` (shadcn) + `@tanstack/react-table` |
| Confirmation dialogs (delete) | `AlertDialog` |
| Dropdown menus (row actions, user avatar menu) | `DropdownMenu` |
| Tag multi-select | `Command` (combobox pattern) |
| Form inputs | `Input`, `Textarea`, `Select`, `Label` |
| Form state management | `Form` (shadcn) + `react-hook-form` + `@hookform/resolvers` |
| Publish control (Draft / Publish / Schedule) | `RadioGroup` |
| OTP digit input (sign-in) | `InputOTP` |
| Schedule date-time picker | `Calendar` + `Popover` |
| Buttons | `Button` (variants: `default`, `outline`, `ghost`, `destructive`) |
| Status badges / tag pills | `Badge` |
| Loading placeholders | `Skeleton` |
| Toast notifications | `Sonner` (already installed in the template) |
| Theme toggle | Existing `theme-toggle.tsx` component |
| Cards (stat cards, comic cards) | `Card` or custom div with Tailwind |
| Pagination | `Pagination` (shadcn) or custom with `Button` variants |
| Breadcrumbs | `Breadcrumb` |
| Tabs (if needed) | `Tabs` |
| User avatar | `Avatar` |
| Mobile filter drawer (browse page) | `Sheet` |
| File upload dropzone | Custom component using native drag-and-drop API + Tailwind styling |
| Image with error fallback | Custom `ImageWithFallback` component (handles `onError`, shows styled fallback) |
| Home page carousel | `Carousel` (shadcn, Embla-based) + `embla-carousel-autoplay` plugin |

### 16.10 Toast Notifications

Use `sonner` (already installed) for all user-facing feedback. Import `toast` from `sonner`.

**Show a success toast after every successful mutation:**
- "Comic created" / "Comic updated" / "Comic deleted"
- "Chapter created" / "Chapter updated" / "Chapter deleted"
- "Segments uploaded (N files)" / "All segments deleted"
- "Tag created" / "Tag deleted"
- "Bookmarked" / "Bookmark removed"

**Show an error toast on failure:**
- "Failed to create comic" (with the error message if available)
- "Upload failed" etc.

**OTP flow feedback:**
- "Code sent to your email" (after sending OTP)
- "Invalid code, please try again" (on verification failure)

**No toast needed for:**
- Read-only actions (navigating, browsing, searching)
- View count increments (invisible to user)
- Reading history upserts (invisible to user)

### 16.11 Image Handling

**Zero layout shift** is a hard requirement. Every image must render inside a container with known dimensions so the layout is stable before the image loads.

#### Cover images (comic cards, comic detail, admin lists)

- Wrap in a fixed aspect ratio container: `aspect-[2/3]` for portrait covers.
- Set `width` and `height` attributes on the `<img>` or use Tailwind's `aspect-ratio` utility on the container.
- While loading: the container shows a `Skeleton` fill matching the aspect ratio (muted background shimmer).
- On error (broken URL or missing image): show a styled fallback — muted background with the comic title text centered, or a generic placeholder icon. Never show the browser's broken image icon.

#### Chapter thumbnails (admin chapter list)

- Small fixed-size container (e.g. 64x96 or similar thumbnail dimensions).
- Same skeleton + fallback pattern as covers.

#### Segment images (chapter reader)

- Dimensions (`width`, `height`) are known from the database and pre-loaded in the route loader.
- The layout algorithm computes `scaledWidth` and `scaledHeight` before any images load (already specified in Section 9.1).
- Each `<img>` must have explicit `width` and `height` attributes set to the scaled dimensions so the browser reserves the exact space.
- While loading: the image container shows a muted rectangle at the computed dimensions (use a `bg-muted` placeholder or `Skeleton` at the exact size).
- Use `loading="lazy"` for off-screen segments (columns not yet scrolled into view).
- On error: show the muted placeholder permanently with a subtle "failed to load" icon — do not collapse the container.

#### Admin upload previews (segment manager, cover/thumbnail dropzones)

- After a file is selected but before upload completes, show a local preview using `URL.createObjectURL`.
- Revoke object URLs on unmount or when replaced (same lifecycle pattern as the prototype viewer).

---

## 17. Key References

| Resource | Path |
|----------|------|
| Existing app scaffold | `wt-web-app/` |
| App README | `wt-web-app/README.md` |
| Agent guidelines | `wt-web-app/AGENTS.md`, `wt-web-app/.agents/` |
| Auth config | `wt-web-app/src/lib/auth/auth.ts` |
| DB schema | `wt-web-app/src/lib/db/schema/` |
| Drizzle config | `wt-web-app/drizzle.config.ts` |
| Prototype layout spec | `webtoon-prototype/docs/WEB_VIEWER_APP.md` |
| Prototype processor spec | `webtoon-prototype/docs/ELECTRON_APP.md` |
