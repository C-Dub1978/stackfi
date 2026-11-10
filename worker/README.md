# worker

The Spring Boot media worker (a Maven module with its own `pom.xml`, `stream.stackfi.worker`). It consumes transcode jobs from RabbitMQ, probes files with ffprobe and runs FFmpeg to produce HLS renditions and thumbnails.

It reads the schema through `common/` but never runs migrations. FFmpeg runs sandboxed: a fixed argument list, CPU and time limits, and no network access.
