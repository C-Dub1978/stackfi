# api

The Spring Boot API (a Maven module with its own `pom.xml`), organised by feature package under `stream.stackfi.api`: `auth`, `catalog`, `library`, `upload`, `playback`, `search`, `playlist` and `admin`.

It owns the database schema (Flyway migrations) and the OpenAPI spec that `web/` generates its client from. Media bytes never pass through it: it checks entitlement and signs URLs.
