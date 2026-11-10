# CLAUDE.md — Stackfi

Stackfi (stackfi.stream) is one app for every kind of media. It has two sources:

- **Catalog:** movies, series, documentaries and music that every signed-in user can stream (public-domain and Creative Commons content only).
- **My Library:** media each user uploads. It's transcoded on upload, private to the user, and both streamable and downloadable.

Books and documents come in Phase 9, and native mobile apps in Phase 10.

**Why this project exists:** the owner is a frontend developer with 8 years' experience who is learning Java, Spring Boot, PostgreSQL and AWS in order to take on backend work at their job. Learning matters more than shipping speed. Read "How to work with me" before doing anything.

---

## How to work with me

### Backend (`api/`, `worker/`, `common/`, SQL, `infra/`): tutor mode

I'm hand-writing the backend to learn it.

- **Do not write or edit backend code unless I explicitly ask** (for example, "write this for me" or "implement X"). Default to explaining, reviewing, giving hints, and pointing to the relevant Spring docs.
- When I ask how to do something, explain the concept and the Spring way of doing it first. Then show a **small** example, not the finished feature.
- When reviewing my code, explain *why* something is wrong or non-idiomatic, and name the Spring or Java concept involved.
- Scaffolding I ask for is fine: config files, `pom.xml` files, Docker Compose and Flyway file names. Business logic I write myself.
- If a request would skip a learning step (for example, "just generate all the entities"), say so briefly, then do what I ask.
- Relate Spring concepts to frontend ones when it helps: DI is like Angular's injector, filters are like HTTP interceptors, and so on.

### Frontend (`web/`): normal pair-programming mode

I'm experienced here. Write and edit code freely, following the conventions below.

### Everywhere

- Prefer boring, well-documented defaults over clever ones.
- The architecture decisions below are settled. Don't relitigate them unless I ask, or you find a concrete problem; if so, say so plainly.
- Ask before adding any new dependency, service or infrastructure piece.
- Keep explanations short and concrete. Don't recap what you did step by step.

---

## Repository layout (one monorepo)

```
stackfi/
├── api/                 Spring Boot API (Maven module, own pom.xml). Owns the DB schema (Flyway) and the OpenAPI spec.
├── worker/              Spring Boot media worker (Maven module, own pom.xml). Consumes jobs, runs FFmpeg. Never runs migrations.
├── common/              Shared job messages, entities, enums (Maven module, plain jar).
├── web/                 Angular 22 app. Capacitor ios/ and android/ shells come in Phase 10.
├── infra/
│   ├── local/           Docker Compose: Postgres, SeaweedFS (S3), RabbitMQ, Redis
│   └── terraform/       AWS resources (Phase 8)
├── docs/                stackfi_platform_planning.md (exported planning doc), decisions/ (decision records)
├── pom.xml              Parent POM: <packaging>pom</packaging>, lists api, worker, common as <modules>
├── mvnw, .mvn/          Maven wrapper (always use ./mvnw, never a global mvn)
└── .github/workflows/   One workflow per app, filtered by path
```

- No monorepo tool (no Nx or Turborepo). Maven builds the Java side, and the Angular CLI and npm build `web/`.
- CI path filters: `api/**` and `common/**` trigger the API build, `worker/**` and `common/**` trigger the worker build, and `web/**` triggers the frontend build.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Language | Java 21 or 25 (LTS) |
| Backend | Spring Boot (latest stable): Web, Data JPA, Security, Validation, Actuator |
| Build | Maven (multi-module, via the Maven wrapper `./mvnw`) |
| Database | PostgreSQL. Flyway migrations, written by hand in SQL |
| Object storage | SeaweedFS (S3-compatible) locally, S3 in production. MinIO stopped publishing images (decision 0003) |
| Queue | RabbitMQ (with a dead-letter queue) |
| Cache | Redis (rate limits, hot data, progress buffer) |
| Media | FFmpeg and ffprobe, run from the Java worker |
| Testing | JUnit 5, Testcontainers (real Postgres, SeaweedFS, RabbitMQ), REST Assured |
| API docs | springdoc-openapi. `web/` generates its typed client from the spec |
| Frontend | Angular 22: standalone, zoneless, signals |
| UI | HMHA design system: `@halfmanhalfape/hmha-ui` and `@halfmanhalfape/hmha-tokens` (v1.2.0+) |
| Playback | hls.js |
| Uploads | Uppy 6, headless (`@uppy/core` + `@uppy/aws-s3`) |
| Native (later) | PWA first, then Capacitor |
| Production | AWS: ECS Fargate, RDS, S3, CloudFront, Route 53, ACM, managed with Terraform |

---

## Architecture: settled decisions

1. **Modular monolith API plus one separate worker.** Organise the API by feature package, not by layer: `stream.stackfi.api.{auth, catalog, library, upload, playback, search, playlist, admin}`. Each package owns its controllers, services, repositories and DTOs.
2. **Media bytes never pass through the API.** Uploads go from the browser straight to object storage using presigned multipart URLs. Playback and downloads use short-lived signed URLs or a CDN. The API only checks entitlement and signs URLs.
3. **The title is separate from its files.** `media_item` is what a title is. `media_asset` and `rendition` are the files that play it. Catalog items and user uploads share one playback path, and only ownership differs.
4. **Entitlement lives in one service method, used everywhere.** Catalog items (`owner_id IS NULL`) can be streamed by any signed-in user, but not downloaded. Library items can be streamed and downloaded by their owner only. Admins can do everything. Never trust an ID from the client: always check ownership in the service layer.
5. **Jobs are asynchronous and idempotent.** The API publishes a job to RabbitMQ, and the worker processes it. Output keys are deterministic (`hls/{assetId}/720p/...`). The worker acknowledges a message only after the outputs are written and the database is updated. Failed jobs retry 3 times, then go to the dead-letter queue.
6. **Auth must support browsers and native apps.** Use a short-lived JWT access token (about 15 minutes) and a rotating refresh token. Store only the refresh token's hash in Postgres. The refresh token travels in an HttpOnly cookie for browsers and in the response body for native apps (Phase 10), which keep it in the device keychain. Design for both from Phase 2.
7. **Resume position is a locator, not a timestamp.** `playback_progress.locator` is JSONB: a timestamp for audio and video, or a Readium-style text position for books.
8. **Media kinds** are `MOVIE, SERIES, EPISODE, DOCUMENTARY, ALBUM, TRACK, BOOK, COMIC`. Books and comics stay unused until Phase 9.
9. **Uppy owns upload behaviour** (validation, multipart, retries, pause and resume). **Spring signs the requests.** Use Uppy's bring-your-own-signer mode, which calls our API. Never give AWS credentials to the browser, and don't use Uppy's Companion server.
10. **The worker is sandboxed.** FFmpeg runs with a fixed argument list (never shell-interpolate user input), CPU and time limits, and no network access.

### Core tables

`app_user`, `refresh_token`, `media_item`, `series_episode`, `album_track`, `person`/`credit`, `genre`/`item_genre`, `media_asset`, `rendition`, `transcode_job`, `library_entry`, `playback_progress`, `playlist`/`playlist_item`.

The full column list is in `docs/stackfi_platform_planning.md`.

### Upload pipeline

1. API: start upload (check quota, create the asset row, return a multipart upload ID)
2. Browser (Uppy): upload the parts straight to storage, with each part signed by the API
3. API: complete the upload, set status to `UPLOADED`, publish the job
4. Worker: probe the file with ffprobe (real type, streams, duration)
5. Worker: transcode (HLS at 1080p, 720p and 480p with H.264 and AAC, AAC audio, thumbnails)
6. Worker: write the renditions and set status to `READY`, or to `FAILED` once retries run out

Asset status runs `UPLOADING` → `UPLOADED` → `PROCESSING` → `READY` | `FAILED`. Never upscale.

---

## Backend conventions

- **Maven layout:** the root `pom.xml` is the parent. It inherits from `spring-boot-starter-parent` and declares shared versions (Java version, Testcontainers BOM) in `<properties>` and `<dependencyManagement>`. Each module's `pom.xml` names the root as its `<parent>`. `api` and `worker` depend on `common` as a normal `<dependency>`. Only `api` and `worker` use `spring-boot-maven-plugin`, because `common` is a plain library jar, not a runnable app.
- Base package: `stream.stackfi`. Modules use `stream.stackfi.api`, `stream.stackfi.worker` and `stream.stackfi.common`.
- Use Java **records** for DTOs and messages. Ask before adding Lombok or MapStruct.
- REST + JSON under `/api`. Return errors as RFC 9457 Problem Details through `@RestControllerAdvice`.
- Never return JPA entities from controllers. Map them to DTOs.
- Use UUID primary keys and `timestamptz` timestamps.
- Name Flyway migrations `V{n}__{snake_case_description}.sql`. Never edit a migration that has already run; add a new one.
- Keep configuration in `application.yml`, with `local`, `test` and `prod` profiles. Secrets come from environment variables, never from the repo.
- Use Testcontainers for integration tests, not mocks of Postgres, S3 storage or RabbitMQ.
- Watch for N+1 queries. Use DTO projections or fetch joins, and check them with `EXPLAIN ANALYZE`.

## Frontend conventions (`web/`)

Full rules live in **`web/CLAUDE.md`**, which loads when you work in `web/`. In short:

- Angular 22: standalone, zoneless, signals, `inject()`, Signal Forms. Component selector prefix `sf-`.
- All UI uses the HMHA design system with tokens only. No other component library without asking.
- Uploads go through a headless Uppy `UploadService`, playback through hls.js, and API calls through the generated OpenAPI client.

---

## Domains

| Host | Purpose |
| --- | --- |
| `stackfi.stream` | Frontend |
| `api.stackfi.stream` | API |
| `media.stackfi.stream` | CDN or media gateway (segments, downloads) |
| `staging.stackfi.stream` / `api.staging.stackfi.stream` | Staging |

These are registered in Route 53 (each in its own hosted zone). Don't create DNS records before Phase 8.

---

## Roadmap

Each phase ends with something demoable. Don't start a phase until the previous one has tests and runs in Docker Compose.

| # | Phase | Learning focus |
| --- | --- | --- |
| 1 | Foundations: Maven multi-module build, Docker Compose, Flyway baseline, health check, CI with Testcontainers | Project structure, profiles, migrations |
| 2 | Accounts: register, login, refresh, logout, roles, `/me` | Spring Security filter chain, JWT, Problem Details |
| 3 | Catalog read side: items, series and episodes, albums and tracks, credits, genres, seed data | JPA relationships, N+1, pagination |
| 4 | Playback of pre-encoded media: S3 storage (SeaweedFS locally), signed URLs, progress, hls.js player | Object storage, HLS |
| 5 | Uploads and transcoding: Uppy multipart, queue, worker, renditions | Messaging, idempotency, retries |
| 6 | Library and downloads: My Library, metadata edits, delete with storage cleanup, quotas | Consistency between the database and storage |
| 7 | Search, playlists, continue watching | Postgres full-text search, Redis caching, query tuning |
| 8 | Production: AWS through Terraform, CDN, dashboards, alerts, backups | Cloud deployment, observability |
| 9 | Books and documents: EPUB, PDF, CBZ/CBR, a reader mode, worker conversion jobs | A second job type, sandboxing |
| 10 | Native apps: PWA, then Capacitor (background audio, offline downloads, keychain tokens) | Non-browser clients |

**Current phase: 0, repo scaffolding.** Update this line as phases complete.

---

## Commands

Fill these in as each part is scaffolded.

```bash
# Local dependencies
docker compose -f infra/local/compose.yaml up -d

# Backend
./mvnw install -DskipTests                     # build all modules once, so api/worker can see common
./mvnw -pl api spring-boot:run -Dspring-boot.run.profiles=local
./mvnw -pl worker spring-boot:run -Dspring-boot.run.profiles=local
./mvnw test                                    # all modules
./mvnw -pl api -am test                        # api plus the modules it depends on (common)

# Frontend
cd web && npm start
cd web && npm test
```

---

## Open questions (don't assume answers; ask)

- Keep original uploads forever, or delete them after transcoding? (This affects "download original".)
- Can users add catalog titles to My Library, or is it uploads only?
- Can playlists mix audio and video?
- What are the per-user storage quota and maximum file size?
- Will the platform be public, or invite-only?

## Reference

- Planning doc: `docs/stackfi_platform_planning.md`
- Decision records: `docs/decisions/` (add one whenever a settled decision or open question is resolved)
- Design system source: https://github.com/C-Dub1978/DS_HalfManHalfApe (its `CLAUDE.md` and `DECISIONS.md` describe its conventions)
- Github repo for Stackfi source: https://github.com/C-Dub1978/stackfi
