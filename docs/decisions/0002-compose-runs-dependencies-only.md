# 0002: Docker Compose runs dependencies only until Phase 8

- **Status:** Accepted
- **Date:** 2026-11-10

## Context

Each phase must "run in Docker Compose" before the next one starts. That could mean either:

- Compose runs only the backing services (Postgres, MinIO, RabbitMQ, Redis), and the API and worker run on the host through `./mvnw` or the IDE.
- Compose also builds and runs the API and worker as containers.

The project's goal is learning Spring Boot, which means a lot of edit, restart and debug cycles. Rebuilding an image for every change slows that loop and makes the debugger harder to attach.

## Decision

Until Phase 8, `infra/local/compose.yaml` runs only the dependencies. The API and worker run on the host with the `local` profile:

```bash
./mvnw -pl api spring-boot:run -Dspring-boot.run.profiles=local
```

Containerising the API and worker, including an FFmpeg-equipped worker image, is part of Phase 8.

## Consequences

- Fast restarts, hot reload and direct debugging from the IDE.
- The `local` profile points at `localhost` ports published by Compose, not at Compose service names.
- Java 25 and FFmpeg must be installed on the host. FFmpeg is needed from Phase 4 onwards.
- "Works in a container" isn't proven until Phase 8, so expect some packaging and config surprises then.
