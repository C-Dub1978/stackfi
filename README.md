# Stackfi

One app for every kind of media. Stackfi streams movies, series, documentaries and music from two sources:

- **Catalog:** public-domain and Creative Commons titles that any signed-in user can stream.
- **My Library:** your own uploads. They're transcoded on upload, private to you, and both streamable and downloadable.

Books and documents arrive in Phase 9, and native iOS and Android apps in Phase 10.

> **Status:** Phase 0, repository scaffolding. Nothing runs yet. See the [roadmap](#roadmap).

## Why it exists

Stackfi is a learning project: a frontend developer building a real backend end to end with Java, Spring Boot, PostgreSQL and AWS. Each feature was chosen because it requires a specific backend skill, such as relational modelling, async jobs, object storage, auth, caching or deployment.

## Architecture

- **A modular monolith API plus one media worker.** The API is a Spring Boot app organised by feature (`auth`, `catalog`, `library`, `upload`, `playback`, `search`, `playlist`, `admin`). The worker consumes jobs from RabbitMQ and runs FFmpeg.
- **Media bytes never pass through the API.** Browsers upload straight to object storage through presigned multipart URLs, and they play media through short-lived signed URLs. The API checks entitlement and signs the URLs.
- **The title is separate from its files.** `media_item` describes what a title is, while `media_asset` and `rendition` are the files that play it. Catalog titles and user uploads share one playback path.
- **Jobs are asynchronous and idempotent.** Output keys are deterministic, a message is acknowledged only after its work is done, and a failed job retries three times before it goes to a dead-letter queue.

The full design is in the [planning doc](docs/stackfi_platform_planning.md), and the reasons behind key choices are in [docs/decisions](docs/decisions/).

## Tech stack

| Layer | Choice |
| --- | --- |
| Backend | Java 21+, Spring Boot (Web, Data JPA, Security, Validation, Actuator) |
| Build | Maven multi-module, run through the `./mvnw` wrapper |
| Database | PostgreSQL with hand-written Flyway migrations |
| Storage | SeaweedFS (S3-compatible) locally, S3 in production |
| Queue | RabbitMQ, with a dead-letter queue |
| Cache | Redis |
| Media | FFmpeg and ffprobe, producing HLS (H.264 and AAC) |
| Frontend | Angular 22 (standalone, zoneless, signals), the [HMHA design system](https://github.com/C-Dub1978/DS_HalfManHalfApe), hls.js, Uppy |
| Testing | JUnit 5, Testcontainers, REST Assured |
| Production | AWS (ECS Fargate, RDS, S3, CloudFront, Route 53), managed with Terraform |

## Repository layout

```
stackfi/
├── api/                 Spring Boot API. Owns the database schema and the OpenAPI spec.
├── worker/              Spring Boot media worker. Runs FFmpeg.
├── common/              Code shared by api and worker: job messages, entities, enums.
├── web/                 Angular 22 frontend.
├── infra/
│   ├── local/           Docker Compose: Postgres, SeaweedFS (S3), RabbitMQ, Redis
│   └── terraform/       AWS infrastructure
├── docs/                Planning doc and decision records
└── .github/workflows/   CI, one workflow per app
```

## Getting started

Setup instructions will appear here once Phase 1 is complete. You'll need:

- Java 21 or later
- Docker with Docker Compose
- Node.js (LTS) and npm

## Roadmap

Each phase ends with something you can demo, and the next phase doesn't start until the current one has tests and runs in Docker Compose.

- [ ] **1. Foundations:** Maven multi-module build, Docker Compose, Flyway baseline, health check, CI with Testcontainers
- [ ] **2. Accounts:** register, login, refresh, logout, roles, `/me`
- [ ] **3. Catalog:** movies, series and episodes, albums and tracks, credits, genres, seed data
- [ ] **4. Playback:** pre-encoded HLS from S3-compatible storage, signed URLs, resume position, the hls.js player
- [ ] **5. Uploads and transcoding:** Uppy multipart uploads, the queue, the worker, renditions
- [ ] **6. Library and downloads:** My Library, metadata edits, deletes with storage cleanup, quotas
- [ ] **7. Search, playlists, continue watching:** Postgres full-text search, Redis caching
- [ ] **8. Production:** AWS through Terraform, CDN, dashboards, alerts, backups
- [ ] **9. Books and documents:** EPUB, PDF, CBZ/CBR, a reader mode
- [ ] **10. Native apps:** a PWA, then Capacitor shells for iOS and Android

## Catalog content

Catalog media comes only from public-domain and Creative Commons sources, such as Blender Studio open movies, the Internet Archive, Free Music Archive and Musopen. Attribution for CC-BY works is stored with each title. User uploads stay private to the uploader.
