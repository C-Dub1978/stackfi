# 0003: SeaweedFS replaces MinIO for local S3 storage

- **Status:** Accepted
- **Date:** 2026-11-10

## Context

The plan used MinIO as the local stand-in for AWS S3, both in Docker Compose and in Testcontainers. While writing the Phase 1 Compose file, we found that MinIO has stopped distributing its community edition as images. The `minio/minio` repository is gone from Docker Hub, and the community edition is source-only and no longer maintained upstream. Testcontainers' MinIO module also defaults to that image.

Production is unaffected, because it uses AWS S3. What we need locally is an S3-compatible server that supports what Stackfi actually uses:

- access-key authentication (SigV4)
- presigned GET and PUT URLs
- multipart uploads (Uppy's `@uppy/aws-s3`)
- per-bucket CORS, so the browser can upload directly

We considered these options:

- **SeaweedFS:** Apache 2.0, maintained for more than 10 years, official images.
- **Garage:** AGPL. It needs extra cluster and key setup before it's usable.
- **RustFS:** a newer drop-in MinIO replacement, less proven.
- **An unofficial MinIO build:** an unvetted third-party image with no upstream security patches.

## Decision

Use SeaweedFS (`chrislusf/seaweedfs`, pinned to a version) as the local S3-compatible store. It runs as a single container (`weed server -s3`), with the S3 API on port 8333.

Credentials come from `S3_ACCESS_KEY` and `S3_SECRET_KEY` in `infra/local/.env`. Compose passes them to SeaweedFS as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. Without them, SeaweedFS accepts any key.

Before choosing it, we verified SeaweedFS 4.53 locally with the AWS CLI:

- It rejects wrong keys.
- A 25 MB upload completed as three parts.
- A presigned GET returned identical bytes.
- Bucket CORS allowed the listed origin and rejected others.

## Consequences

- Application code talks to it through the standard AWS SDK S3 client, with an endpoint override in the `local` and `test` profiles, so the same code works against S3 in production.
- Use path-style URLs locally (`http://localhost:8333/bucket/key`), and keep the region set to `us-east-1`.
- Integration tests run SeaweedFS through Testcontainers' `GenericContainer`, not the MinIO module.
- SeaweedFS has no MinIO-style web console. Inspect buckets with the AWS CLI (`--endpoint-url http://localhost:8333`).
- Behaviour can still differ from real S3 in edge cases, such as error codes, CORS details and checksum headers. Phase 8 testing against real S3 will catch those.
- Decision 0002 and the planning doc still mention MinIO. This record supersedes those mentions.
