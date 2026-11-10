# Stackfi — Planning Doc

Oct 18, 2026 · @Christopher Wilson

## Vision & scope

One app, **Stackfi**, that streams video and audio from two sources: a shared **catalog** everyone can stream, and a **personal library** each user fills with their own uploads, which they can stream *and* download.

The real goal is learning the backend stack end to end. Every feature below is picked because it forces you into a specific backend skill: relational modeling, async jobs, file storage, auth, caching, background workers, observability and deployment.

- **Catalog** — movies, series (seasons → episodes), documentaries, albums, tracks. Read-only for users, managed by an admin role. Stream only.
- **My Library** — anything a user uploads. Transcoded into streamable formats, private by default. Stream + download (original or transcoded).
- **Unified experience** — one search, one "continue watching/listening", one player shell that switches between video and audio modes.

Out of scope for v1: payments, DRM, recommendations ML, social features, native mobile apps.

## Features

Ship the MVP column first; each later tier adds one new backend concept rather than more of the same.

| &#32;Area | MVP | v2 | Stretch |
| --- | --- | --- | --- |
| Accounts | Sign up, log in, refresh tokens, profile | Email verification, password&#32; | OAuth (Google/GitHub), multiple profiles per account |
| Catalog | Browse movies, series, albums; detail pages | Genres, collections, trending rows | Admin CMS with bulk import |
| Playback | HLS video + audio streaming, resume position | Subtitles (VTT), audio track switching | Adaptive bitrate tuning, Chromecast |
| My Library | Upload video/audio, see processing status, stream | Edit metadata, auto-fetch posters/album art | Folder import, duplicate detection by hash |
| Downloads | Download original file | Download a chosen transcoded quality | Zip a whole album/season |
| Transcoding | One video ladder (1080/720/480), one audio rendition | Thumbnails, sprite sheet for scrubbing previews | GPU encoding, priority queues |
| Discovery | Search by title | Full-text search across catalog + library | Recommendations from watch history |
| Social / lists | Watchlist / favorites | Playlists for audio and video | Shared playlists |
| Limits | Per-user storage quota | Upload rate limiting | Plans/tiers |
| Admin | Seed script for catalog | Admin role + job dashboard | Moderation, reports |

**Supported upload formats (MVP):** video — MP4, MKV, MOV, AVI, WebM; audio — MP3, FLAC, WAV, AAC/M4A, OGG, Opus. FFmpeg reads all of these, so the allow-list is a policy choice, not a technical limit.

## System architecture

Build a **modular monolith API plus one separate transcode worker**. Two deployables is enough to learn messaging and independent scaling without drowning in microservice plumbing.

&#91;embedded content: system architecture · 8 components\]

Uploads and playback go straight between the browser and storage/CDN; the API issues the signed URLs, and the worker is the only thing that runs FFmpeg.

Inside the API, organise by feature package rather than by layer: `auth`, `catalog`, `library`, `upload`, `playback`, `search`, `playlist`, `admin`. Each owns its controllers, services, repositories and DTOs, which makes a later split into services straightforward.

### Architecture map

Every moving part in Stackfi, what it does, where its code lives, and what it becomes on AWS. Each row is one drill-down: pick a row, plan it in detail, scaffold its repo or module, then build it.

| # | Component | Its job | Folder | Local dev | AWS / distribution | Phase |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Local dev environment | Runs every dependency with one command | `infra/local` | Docker Compose | — | 1 |
| 2 | Database | System of record: users, titles, assets, jobs, progress | `api/` (Flyway migrations) | Postgres container | RDS for PostgreSQL | 1 |
| 3 | API: auth | Sign up, login, JWT + refresh tokens, roles | `api/` | Spring Boot app | ECS Fargate | 2 |
| 4 | API: catalog | Browse and detail for movies, series, albums (books in Phase 9) | `api/` | — | — | 3 |
| 5 | Object storage | Holds originals, HLS output, thumbnails, ebooks | — | MinIO container | S3 | 4 |
| 6 | API: playback & entitlements | Who may play/read/download what; signed URLs; resume locator | `api/` | — | CloudFront signed URLs | 4 |
| 7 | Web client | Angular 22 UI; video and audio player, reader in Phase 9 | `web/` | `ng serve` | S3 + CloudFront at `stackfi.stream` | 2–8 |
| 8 | Message queue | Carries transcode and conversion jobs; dead-letter queue | — | RabbitMQ container | Amazon MQ (RabbitMQ) or SQS | 5 |
| 9 | API: library & upload | Presigned uploads, quotas, enqueue jobs, My Library | `api/` | — | — | 5–6 |
| 10 | Media worker | FFmpeg → HLS renditions; ebook conversion and cover extraction (Phase 9) | `worker/` | Docker container | ECS Fargate or EC2 Spot | 5, 9 |
| 11 | Shared code | Job messages, entities, enums used by API and worker | `common/` | Maven module | — | 5 |
| 12 | Cache | Rate limits, hot data, progress buffer | — | Redis container | ElastiCache | 7 |
| 13 | API: search & playlists | Full-text search, playlists, continue watching/listening/reading | `api/` | — | — | 7 |
| 14 | Media delivery (CDN) | Serves segments and downloads close to users | — | MinIO direct | CloudFront at `media.stackfi.stream` | 8 |
| 15 | DNS & TLS | Domain records and HTTPS certificates | `infra/terraform` | — | Route 53 + ACM | 8 |
| 16 | Infrastructure as code | Defines every AWS resource reproducibly | `infra/terraform` | — | Terraform (or AWS CDK) | 8 |
| 17 | CI/CD | Test, build images, deploy | `.github/workflows` | — | GitHub Actions → ECR → ECS | 1, 8 |
| 18 | Observability | Metrics, logs, alerts | `api/`, `worker/`, `infra/` | Prometheus + Grafana | CloudWatch | 8 |
| 19 | Native shells | Wrap the built web app as iOS and Android apps; native plugins for background audio and offline downloads | `web/ios`, `web/android` (Capacitor) | Xcode, Android Studio | App Store, Google Play | 10 |

**Repository: one monorepo, `stackfi`.** The API and worker share code and change together, so one repo means one commit per change instead of coordinated releases. Any folder can be split into its own repo later with its history intact.

```
stackfi/
├── api/                 Spring Boot API (Maven module, own pom.xml)
├── worker/              Spring Boot media worker (Maven module, own pom.xml)
├── common/              Shared job messages, entities, enums (Maven module, plain jar)
├── web/                 Angular 22 app; Capacitor ios/ and android/ shells (Phase 10)
├── infra/
│   ├── local/           Docker Compose: Postgres, MinIO, RabbitMQ, Redis
│   └── terraform/       AWS resources (Phase 8)
├── docs/
├── pom.xml              Parent POM: lists api, worker, common as <modules>
├── mvnw, .mvn/          Maven wrapper
└── .github/workflows/   One workflow per app, filtered by folder
```

- `api/` owns the database schema (Flyway) and the REST API's OpenAPI spec.
- `worker/` reads the same schema through `common/` but never runs migrations.
- `web/` generates its typed API client from the OpenAPI spec.
- CI path filters keep pipelines separate: `api/**` and `common/**` trigger the API build, `web/**` triggers the frontend build, and so on.

**Each drill-down covers the same six things:** responsibilities and boundaries; interfaces (endpoints, messages, tables); key decisions; concepts to learn first; scaffold checklist; and a "done when" definition.

## Tech stack

Stick to boring, well-documented defaults so the hard parts are the domain, not the tooling.

| Layer | Choice | Why it fits a learning project |
| --- | --- | --- |
| Language / runtime | Java 21 or 25 (LTS) | Records, sealed types, virtual threads for I/O-heavy endpoints |
| Framework | Spring Boot (latest stable) | Web, Data JPA, Security, Validation, Actuator in one coherent stack |
| Build | Maven (multi-module, via the \`./mvnw\` wrapper) | Matches the Spring tutorials and most Spring teams; a parent POM ties api, worker and common together |
| Database | PostgreSQL | Relational core, JSONB for loose metadata, `tsvector` full-text search |
| Migrations | Flyway | Versioned SQL you write by hand — best way to learn Postgres |
| Object storage | MinIO locally, S3 / Cloudflare R2 in prod | S3 API everywhere; presigned URLs keep big files off your app servers |
| Queue | RabbitMQ (or Postgres `SKIP LOCKED` to start) | Decouples uploads from transcoding; retries and dead-letter queues |
| Transcoding | FFmpeg + ffprobe, driven from a Java worker | Industry standard; you learn process management and HLS packaging |
| Cache / sessions | Redis | Rate limits, hot catalog rows, playback-position write buffering |
| Search | Postgres FTS first, OpenSearch later | Avoid a second datastore until you feel the pain |
| Frontend | Angular 22 + hls.js + HMHA design system | Your strength — keep it thin so time goes to the backend |
| Testing | JUnit 5, Testcontainers, REST Assured | Real Postgres/MinIO/RabbitMQ in tests, not mocks |
| Observability | Actuator, Micrometer → Prometheus + Grafana, structured JSON logs | See queue depth, transcode time, error rates |
| Local dev | Docker Compose | One command brings up Postgres, MinIO, RabbitMQ, Redis |

### Design system

Stackfi's UI uses the **Half Man Half Ape (HMHA) design system**: `@halfmanhalfape/hmha-tokens` (pure-CSS design tokens) and `@halfmanhalfape/hmha-ui` (standalone, zoneless, signal-based Angular components on Angular CDK). Source: [C-Dub1978/DS\_HalfManHalfApe](https://github.com/C-Dub1978/DS_HalfManHalfApe). Stackfi is its first real-world consumer.

**Before Phase 2 (blocker)**

- [x] Add `'22'` to the CI install matrix in `.github/workflows/ci.yml` and get it green.
- [x] Widen the `hmha-ui` peer range from `<22.0.0` to `<23.0.0` (and in `CLAUDE.md`), then publish 1.1.0.
- [ ] Update the README's Status section, which still describes Phase 2 as in progress.

**Components Stackfi needs that the library doesn't have yet**

- [ ] Slider — seek bar and volume (highest priority)
- [ ] Progress bar — upload and transcode status
- [ ] Spinner and skeleton — loading states for catalog rows
- [ ] Avatar — user profiles
- [ ] Badge — "new", "processing", "failed" labels

**Timebox rule:** when Stackfi needs something the library lacks, give it a fixed timebox. If it runs over, use Angular CDK or Angular Material for that piece and move on — backend learning stays the priority. The player controls themselves are Stackfi code, not library code.

## Data model

The key design move: separate **what a title is** (`media_item`) from **the files that play it** (`media_asset` + `rendition`). Catalog and user uploads then share one playback path; only ownership differs.

| Table | Key columns | Notes |
| --- | --- | --- |
| `app_user` | id (UUID), email (unique), password\_hash, display\_name, role, storage\_quota\_bytes, created\_at | Role enum: USER, ADMIN |
| `refresh_token` | id, user\_id, token\_hash, expires\_at, revoked\_at | Store a hash, never the raw token |
| `media_item` | id, kind, title, description, release\_year, runtime\_ms, owner\_id (nullable), visibility, metadata JSONB, search\_vector | kind: MOVIE, SERIES, EPISODE, DOCUMENTARY, ALBUM, TRACK, BOOK, COMIC. owner\_id NULL = catalog |
| `series_episode` | episode\_item\_id, series\_item\_id, season\_number, episode\_number | Unique on (series, season, episode) |
| `album_track` | track\_item\_id, album\_item\_id, disc\_number, track\_number | Same pattern for audio |
| `person` / `credit` | person: id, name; credit: item\_id, person\_id, role (ACTOR, DIRECTOR, ARTIST…) | Many-to-many |
| `genre` / `item_genre` | genre: id, name; join table |  |
| `media_asset` | id, item\_id, owner\_id, original\_key, original\_filename, mime\_type, size\_bytes, sha256, probe JSONB, status | status: UPLOADING, UPLOADED, PROCESSING, READY, FAILED |
| `rendition` | id, asset\_id, type (HLS\_VIDEO, HLS\_AUDIO, THUMBNAIL, DOWNLOAD\_MP4…), height, bitrate\_kbps, codec, manifest\_key, size\_bytes | One row per output |
| `transcode_job` | id, asset\_id, status, attempts, worker\_id, progress\_pct, error, started\_at, finished\_at | Your job queue history |
| `library_entry` | user\_id, item\_id, added\_at | What appears in "My Library" (also lets users save catalog titles) |
| `playback_progress` | user\_id, item\_id, locator JSONB, progress\_pct, completed, updated\_at | PK (user\_id, item\_id); locator holds a timestamp (audio/video) or a Readium-style text position (books); powers "continue watching" |
| `playlist` / `playlist_item` | playlist: id, owner\_id, name, kind; item: playlist\_id, item\_id, position | Mixed audio/video allowed or not — your call |

Postgres skills this exercises: UUID keys, enums or check constraints, partial indexes (`WHERE owner_id IS NULL` for catalog queries), a GIN index on `search_vector`, JSONB for raw ffprobe output, and row-level ownership checks in every query.

## Upload & transcoding pipeline

This is the heart of the project and where you'll learn the most: large-file handling, async jobs, retries and process control.

&#91;embedded content: upload & transcoding pipeline · 6 steps\]

The API only touches small JSON; the worker owns every FFmpeg run and reports progress back through the job row.

**Output ladder (start simple, tune later)**

| Rendition | Codec | Approx. bitrate | Applies to |
| --- | --- | --- | --- |
| 1080p HLS | H.264 + AAC | \~5,000 kbps | Video sources ≥ 1080p |
| 720p HLS | H.264 + AAC | \~2,800 kbps | All video |
| 480p HLS | H.264 + AAC | \~1,200 kbps | All video |
| Audio HLS | AAC | 256 kbps | Songs and audio-only |
| Thumbnails | JPEG | — | Poster frame + scrub sprite (v2) |

Never upscale: skip rungs above the source resolution. H.264/AAC plays everywhere; add HEVC or AV1 later as a learning exercise.

**Worker rules that keep it robust**

- **Idempotent jobs:** output keys are deterministic (`hls/{assetId}/720p/...`), so a retried job overwrites rather than duplicates.
- **Progress:** parse FFmpeg's `-progress pipe:1` output and update `progress_pct` every few seconds.
- **Timeouts and limits:** kill runaway processes; cap concurrent jobs per worker to the CPU count.
- **Ack after success:** acknowledge the message only after outputs are written and the DB is updated.
- **Cleanup:** a scheduled job removes abandoned multipart uploads and orphaned files.

## Streaming & downloads

The API never pipes media bytes itself; it checks entitlement, then hands the client short-lived signed URLs to object storage or a CDN.

**Entitlement rule (one service method, used everywhere):**

- Catalog item (`owner_id IS NULL`) → any logged-in user may **stream**; no download.
- Library item → only the owner may **stream and download**.
- Admin → everything, for support and debugging.

**Streaming flow**

1. Client calls `GET /api/playback/{itemId}`.
2. API checks entitlement and that the asset is READY.
3. API returns the master `.m3u8` URL plus available qualities, subtitles and the saved resume position.
4. Segments are fetched via signed URLs or signed cookies (CloudFront, R2 + Worker, or a small Spring "segment gateway" while learning).
5. Player posts progress every \~10 s to `PUT /api/progress/{itemId}`; buffer in Redis and flush to Postgres in batches.

**Download flow**

1. `POST /api/library/{itemId}/download?variant=original|1080p|720p|audio-320`.
2. API verifies ownership and returns a presigned GET URL valid for \~15 minutes with a `Content-Disposition: attachment` filename.
3. Optional stretch: an MP4 "download rendition" built on demand, since HLS segments aren't a single file.

**Range requests:** if you serve any progressive MP4 or audio file directly from Spring, implement HTTP `Range` (206 Partial Content) yourself once — it's a great exercise, then move it behind storage/CDN.

## Auth & security

Build auth yourself with Spring Security first; it's where most of the framework's "magic" lives, and understanding it pays off everywhere else.

- **Tokens:** short-lived JWT access token (\~15 min) + rotating refresh token in an HttpOnly, Secure, SameSite cookie. Refresh tokens hashed in Postgres so you can revoke them.
- **Passwords:** BCrypt or Argon2 via Spring's `PasswordEncoder`.
- **Authorization:** method security (`@PreAuthorize`) plus an ownership check in the service layer — never trust an ID from the client.
- **Uploads:** validate by sniffing the file with ffprobe, not by extension or MIME header. Enforce max size and per-user quota before issuing a presigned URL.
- **FFmpeg safety:** run workers with no network access, a fixed argument list (never interpolate user strings into a shell), CPU/time limits, and in their own container.
- **Rate limiting:** Redis token bucket on login, upload-init and search (Bucket4j works well).
- **Signed media URLs:** short expiry, scoped to one user/asset, so links can't be shared broadly.
- **Later:** swap hand-rolled auth for Keycloak or Spring Authorization Server to learn OAuth2/OIDC properly.

## API surface

REST + JSON under `/api`, documented with springdoc-openapi so your frontend can generate a typed client.

| Method & path | Purpose | Auth |
| --- | --- | --- |
| `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout` | Account lifecycle | Public / cookie |
| `GET /me` | Current user, quota used | User |
| `GET /catalog?kind=&genre=&page=` | Browse catalog | User |
| `GET /items/{id}` | Detail (title, credits, seasons/tracks) | User + entitlement |
| `GET /search?q=` | Search catalog + own library | User |
| `POST /uploads` | Start upload → asset id + presigned (multipart) URLs | User |
| `POST /uploads/{assetId}/complete` | Confirm parts uploaded, enqueue transcode | Owner |
| `GET /uploads/{assetId}` | Processing status + progress % | Owner |
| `GET /library` | My Library list | User |
| `PATCH /library/items/{id}` / `DELETE` | Edit metadata / remove (deletes files) | Owner |
| `GET /playback/{itemId}` | Manifest URL, qualities, resume position | User + entitlement |
| `PUT /progress/{itemId}` | Save position | User |
| `GET /continue` | Continue watching/listening row | User |
| `POST /library/items/{id}/download` | Presigned download URL | Owner |
| `GET/POST /playlists`, `/playlists/{id}/items` | Playlists | Owner |
| `GET /admin/jobs`, `POST /admin/catalog` | Job dashboard, catalog management | Admin |

For live processing status, start with polling; later add Server-Sent Events (`GET /uploads/events`) fed by job-progress messages.

## Catalog content (legal sourcing)

Fill the "Netflix/Spotify" half with public-domain and Creative Commons media so you can deploy publicly without licensing problems; keep attribution fields in `metadata` for CC-BY works.

| Source | Media | Licence to check |
| --- | --- | --- |
| [Blender Studio open movies](https://studio.blender.org/films/) | Short films (Big Buck Bunny, Sintel, Tears of Steel…) | CC-BY |
| [Internet Archive feature films](https://archive.org/details/feature_films) | Public-domain movies, old TV, documentaries | Per item |
| [Free Music Archive](https://freemusicarchive.org) | Albums and tracks | Per track, mostly CC |
| [Musopen](https://musopen.org) | Classical recordings | Public domain / CC |
| [Jamendo](https://www.jamendo.com) | Independent music | CC; check commercial terms |

For metadata and artwork on user uploads (v2), look at [TMDB](https://www.themoviedb.org) for film/TV and [MusicBrainz](https://musicbrainz.org) + Cover Art Archive for music. Both have free APIs with attribution rules.

User uploads: add a terms checkbox that users only upload media they have the right to store, and keep libraries private — that's what keeps the "bring your own media" side reasonable.

## Roadmap

Ten phases, each ending in something you can demo. Don't start a phase until the previous one has tests and runs in Docker Compose.

1. **Foundations** — Spring Boot project, Docker Compose (Postgres, MinIO, RabbitMQ, Redis), Flyway baseline, health check, CI running tests with Testcontainers.
   - Learn: project structure, profiles, config properties, migrations.
2. **Accounts** — register/login/refresh/logout, roles, `/me`.
   - Learn: Spring Security filter chain, JWT, password hashing, exception handling with `@ControllerAdvice` + Problem Details.
3. **Catalog read side** — `media_item`, series/episodes, albums/tracks, credits, genres; seed script; paginated browse + detail endpoints.
   - Learn: JPA relationships, N+1 queries, DTO projections, pagination.
4. **Playback of pre-encoded media** — hand-encode a few files to HLS with FFmpeg, upload to MinIO, serve via `/playback`; progress tracking; Angular player with hls.js.
   - Learn: object storage, presigned URLs, HLS basics.
5. **Uploads + transcoding** — presigned multipart upload, ffprobe validation, queue, worker, renditions, status polling.
   - Learn: async messaging, idempotency, retries, process management.
6. **Library + downloads** — My Library, metadata edit, delete (with storage cleanup), quotas, download URLs.
   - Learn: transactional consistency between DB and storage.
7. **Search, playlists, continue watching** — Postgres FTS, playlists, Redis-buffered progress.
   - Learn: indexing, caching, query tuning with `EXPLAIN ANALYZE`.
8. **Production** — deploy, CDN, metrics dashboards, alerts, backups.
   - Learn: containers in prod, secrets, observability.
9. **Books & documents** — EPUB, PDF and comics (CBZ/CBR) first; public-domain catalog from Project Gutenberg / Standard Ebooks; worker conversion jobs (unlocked MOBI/AZW3 → EPUB via Calibre), cover and table-of-contents extraction; reader mode with epub.js or Readium and pdf.js; reading position stored as a locator. Office documents (DOCX/ODT) only behind a sandboxed converter. M4B audiobooks ride the existing audio pipeline.
   - Learn: a second job type on the same queue, polymorphic content in one data model, sandboxing untrusted conversions.
10. **Native apps** — PWA first (`@angular/pwa`), then Capacitor shells for iOS and Android; native plugins for background audio with lock-screen controls and offline downloads; token storage in the device keychain instead of cookies.
    - Learn: CORS and auth for non-browser clients, app store builds and review.

## Deployment & ops

Start on a single VPS with Docker Compose; only move to Kubernetes if you specifically want to learn it. Transcoding is CPU-hungry, so the worker is what you size and scale.

| Option | Shape | Good for |
| --- | --- | --- |
| Single VPS (Hetzner, DigitalOcean) | Compose: API, worker, Postgres, Redis, RabbitMQ, Caddy; media in R2/S3 | Cheapest, simplest first deploy |
| Managed pieces | API + worker on Fly.io/Render/ECS; managed Postgres; S3 + CloudFront or R2 | Learning cloud services without ops burden |
| Kubernetes | Separate deployments, HPA on worker from queue depth (KEDA) | If K8s is itself a learning goal |

### Domain & subdomains

Registered domain: **stackfi.stream** (Route 53). Media gets its own subdomain so the CDN can change later without touching the API, and auth cookies stay off media requests.

| Subdomain | Points to | Phase |
| --- | --- | --- |
| `stackfi.stream` | Frontend app | 8 |
| `api.stackfi.stream` | Spring Boot API | 8 |
| `media.stackfi.stream` | CDN / media gateway (HLS segments, downloads) | 8 |
| `staging.stackfi.stream` | Staging environment (staging API under `api.staging.stackfi.stream`) | 8 |

Ops checklist:

- [ ] Build images with Spring Boot buildpacks or a multi-stage Dockerfile; worker image includes FFmpeg.
- [ ] Secrets via environment / secret manager, never in the repo.
- [ ] HTTPS via Caddy or a managed load balancer.
- [ ] Nightly `pg_dump` (or managed backups) + storage lifecycle rules for orphaned uploads.
- [ ] Dashboards: request latency, queue depth, transcode duration, failed jobs, storage used.
- [ ] Alerts on dead-letter queue growth and disk/CPU on worker hosts.
- [ ] GitHub Actions: test → build image → deploy on main.

## Open questions

- [ ] Monolith with a separate worker, or several services from day one? (Recommendation: modular monolith + worker; split later.)
- [ ] RabbitMQ from the start, or a Postgres-backed queue first?
- [ ] Keep originals forever, or delete after transcoding to save storage? (Downloads of "original" depend on this.)
- [ ] HLS only, or also DASH? (HLS alone covers every modern browser via hls.js.)
- [ ] Can a user add catalog titles to "My Library", or is My Library uploads-only?
- [ ] Should playlists mix audio and video?
- [ ] Storage quota per user, and max single-file size?
- [ ] Is this ever public, or invite-only? That decides how much moderation and abuse protection you need.
