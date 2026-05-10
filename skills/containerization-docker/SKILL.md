---
name: containerization-docker
description: "Build Docker images and run local Compose stacks for Spring Boot 4 services — multi-stage JVM builds, layer caching, distroless bases, Compose for local dev. Read before writing or auditing a Dockerfile or compose.yml. Triggers: Dockerfile, multi-stage build, FROM eclipse-temurin, FROM bellsoft/liberica-runtime-container, layered jar, paketo, buildpacks, docker compose, compose.yml, .dockerignore, distroless, healthcheck, USER 1000."
version: 0.2.0
license: Apache-2.0
---

# Containerization — Docker

**Signals**: `Dockerfile`, `compose.yml` / `docker-compose.yml`, `.dockerignore`, build args for layered jars, Cloud Native Buildpacks (`./mvnw spring-boot:build-image`), Paketo references.

## Tested With

- Docker 24+, BuildKit enabled
- Spring Boot 4.x (layered jar by default)
- Eclipse Temurin 25, BellSoft Liberica Runtime Container 25
- Cloud Native Buildpacks via `spring-boot-maven-plugin`

## Do NOT Use This Skill When

- Configuring secrets / runtime config → use `core-setup`
- Configuring app-level health probes (Actuator) → use `spring-boot-actuator`

## When to Read References

| Situation | Read |
|-----------|------|
| Multi-stage Dockerfile for layered Spring Boot jar | `references/docker-deployment.md` |
| `spring-boot:build-image` (Buildpacks) vs hand-written Dockerfile trade-offs | `references/docker-deployment.md` |
| Local dev with Compose: Postgres, Kafka, OTel collector | `references/docker-deployment.md` |
| Image hardening: distroless, non-root user, read-only FS, healthchecks | `references/docker-deployment.md` |
| `.dockerignore` for Maven projects | `references/docker-deployment.md` |

## Anti-patterns

- Don't build images by `COPY target/*.jar app.jar` over a fat JAR — use layered jar extraction so Docker can cache deps.
- Don't run as root (`USER 1000:1000` minimum in production images).
- Don't bake secrets into image layers — pass via env at runtime or use Docker secrets.
- Don't omit `HEALTHCHECK` for non-orchestrated runs; for K8s rely on probes from `spring-boot-actuator`.
