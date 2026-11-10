# 0001: Use Maven, not Gradle, for the Java build

- **Status:** Accepted
- **Date:** 2026-11-09

## Context

The `api`, `worker` and `common` projects need a multi-project Java build. Spring Initializr supports both Maven and Gradle, and both are well supported by Spring Boot.

The planning doc chose Maven, but for a while `CLAUDE.md` said Gradle with the Kotlin DSL. The two had to be reconciled before any build files were written.

The project's main goal is learning Spring Boot. The video course the owner is following uses Maven and `pom.xml` throughout, as do most Spring guides and many Spring teams.

## Decision

Use a Maven multi-module build, always run through the Maven wrapper (`./mvnw`):

- The root `pom.xml` is the parent (`<packaging>pom</packaging>`). It lists `api`, `worker` and `common` as `<modules>`, inherits from `spring-boot-starter-parent`, and manages shared versions such as the Java version and the Testcontainers BOM.
- Each module has its own `pom.xml` that names the root as its `<parent>`.
- `api` and `worker` depend on `common` as a normal `<dependency>`. Only they use `spring-boot-maven-plugin`, because `common` is a plain library jar.

## Consequences

- Course material, Spring guides and Stack Overflow answers apply to the build directly, with no translation from Gradle.
- POM files are more verbose than Gradle build scripts, and custom build logic is harder to express. Stackfi doesn't need much of that.
- Running one module requires the modules it depends on: use `-pl api -am`, or run `./mvnw install` first so `common` is available.
- CI workflows call `./mvnw`, and `.gitignore` covers Maven's `target/` and plugin leftovers instead of Gradle's `build/` and `.gradle/`.
- Switching to Gradle later would need a new decision record and a rewrite of the build files. No application code would change.
