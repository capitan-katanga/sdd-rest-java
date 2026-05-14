# sdd-rest-java

Claude Code plugin for **Spec-Driven Development of modern Java REST services** on **Spring Boot 4.x** (GA Nov 2025, Java 25, Jakarta EE 11, Spring Framework 7).

Flow-agnostic: integrates with Spec Kit, Kiro, or Claude Code natively. Provides skills, agents, and slash commands; any orchestrating SDD flow consumes them.

## Scope

- REST patterns, OpenAPI/SpringDoc, Actuator, caching, resilience (Resilience4j), event-driven, saga
- Persistence: Spring Data JPA, Spring Data Neo4j
- Security: Spring Security 7 + JWT/OAuth2 + Problem Details (RFC 7807)
- Spring Cloud microservices: discovery & config, gateway, Kafka, HTTP Interface clients, async/concurrency
- Containerization: Docker (multi-stage)
- Observability: Logback, MDC, correlation IDs, OpenTelemetry tracing
- Documentation: OpenAPI annotations on controllers; JEP 467 `///` Markdown Javadoc on services/repositories
- Testing: JUnit 5, Mockito, AssertJ, Testcontainers, WireMock — JPA/MVC/Security/WebFlux/WebSocket slices

**Out of scope:** AI/ML, LangChain4j, Spring AI MCP, vector stores.

## Contents

- **41 skills** (auto-loaded by triggers in their frontmatter `description`)
- **7 agents** (backend dev, code review, unit testing, refactor, security, architecture review, documentation)
- **11 commands** with `sddrest.*` prefix
- **4 rules** (error handling, language best practices, naming, project structure)

## Loading model at a glance

| Type | When loaded | Where it lives |
|---|---|---|
| Rules | Always, every conversation | `rules/*.md` |
| Agents | By name, when invoked via Task tool, a slash command, or another skill/agent | `agents/*.md` |
| Commands | When the user types `/sdd-rest-java:sddrest.<name>` | `commands/sddrest.*.md` |
| Skills | Automatically by triggers — Claude reads only the `description` frontmatter and loads the body if it matches the conversation | `skills/<name>/SKILL.md` |

The skill `description` is the filter. References inside a skill's `references/` directory load only when the SKILL.md cites them explicitly.

## Usage

Inside any Spring Boot 4.x project, three modes — least to most explicit:

1. **Implicit (the main one).** Just describe the work: *"add a REST `/orders` endpoint with CRUD + tests"*. Claude detects triggers and loads `spring-boot-rest-api-standards`, `spring-boot-crud-patterns`, `spring-data-jpa`, `spring-mvc-testing`, `spring-jpa-testing` automatically.

2. **Slash commands** for predefined flows:
   ```
   /sdd-rest-java:sddrest.generate-crud Order
   /sdd-rest-java:sddrest.write-unit-tests src/main/java/.../OrderService.java
   /sdd-rest-java:sddrest.security-review
   /sdd-rest-java:sddrest.upgrade-dependencies
   ```

3. **Specific agents.** Ask explicitly: *"use the `spring-boot-code-review-expert` to review this PR"* or *"have the `java-security-expert` audit this auth code"*. Each agent has its own model and tool allowlist.

### SDD integration

The plugin is flow-agnostic — combine it with whichever SDD flow you use:
- **Spec Kit:** `/specify` → `/plan` → `/tasks` → `/implement` (the plugin engages on `/implement`)
- **Kiro:** `requirements/design/tasks.md` feed `spring-boot-backend-development-expert`
- **Native:** Claude Code plan mode + automatic skill loading

## Documentation

- [INSTALL.md](INSTALL.md) — installation, verification, smoke test
- [.claude-plugin/README.md](.claude-plugin/README.md) — `plugin.json` manifest format
- [agents/README.md](agents/README.md) — agent roster, frontmatter, add/remove
- [commands/README.md](commands/README.md) — slash command roster, frontmatter, add/remove
- [rules/README.md](rules/README.md) — always-on rules, scoping with `paths:`, add/remove
- [skills/README.md](skills/README.md) — skill loading model, triggers, references, add/remove

## Versioning

Tracks Spring Boot 4.x line. Boot 3.x is **not** a target — references to legacy patterns live only in migration sections.
