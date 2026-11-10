# infra/local

Docker Compose for local development. It runs only the backing services. The API and worker run on your machine through `./mvnw` (see [decision 0002](../../docs/decisions/0002-compose-runs-dependencies-only.md)).

## First run

```bash
cp infra/local/.env.example infra/local/.env   # local credentials, git-ignored
docker compose -f infra/local/compose.yaml up -d --wait
```

`--wait` returns once every service passes its health check.

## Services

| Service | Image | Address |
| --- | --- | --- |
| Postgres | `postgres:18-alpine` | `localhost:5432` |
| RabbitMQ | `rabbitmq:4.4-management-alpine` | AMQP at `localhost:5672`, admin UI at http://localhost:15672 |
| Redis | `redis:8.10-alpine` | `localhost:6379` (no password locally) |
| Object storage | `chrislusf/seaweedfs:4.53` | S3 API at http://localhost:8333 (region `us-east-1`). It replaces MinIO; see [decision 0003](../../docs/decisions/0003-seaweedfs-replaces-minio-locally.md) |

The credentials come from `.env`. Ports are bound to `127.0.0.1` only.

## Everyday commands

```bash
docker compose -f infra/local/compose.yaml ps        # status and health
docker compose -f infra/local/compose.yaml logs -f postgres
docker compose -f infra/local/compose.yaml stop      # stop, keep data
docker compose -f infra/local/compose.yaml down -v   # remove containers AND all data
```

If you change a password in `.env` after the first run, Postgres and RabbitMQ keep the old one, because it's stored in their data volumes. Run `down -v` to start fresh.
