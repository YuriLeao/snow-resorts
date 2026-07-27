# ADR 0001 — Backend foundation decisions

Status: Accepted · Date: 2026-06 · Updated: 2026-07

## Context

Bootstrapping the Snow Resorts backend as a **polyrepo** (5 selective microservices + shared
libs in independent Git repositories), targeting Java 25 and a current Spring Boot release
train.

## Decisions

1. **Spring Boot 4.1.0.** Services and `snow-resorts-shared` use
   `spring-boot-starter-parent` **4.1.0** (Java 25). Boot 4 brings the current Framework /
   Jackson 3 / starter layout used in this codebase (e.g. `spring-boot-starter-webmvc`,
   `spring-boot-starter-security-oauth2-resource-server`, `spring-boot-starter-flyway`).
   The version lives in each module’s `<parent>` so it is trivial to bump in lockstep.

2. **No Lombok.** Lombok’s annotation processor historically lags new JDKs and can break the
   build on Java 25. We use explicit constructors (constructor injection), Java `record`s for
   DTOs/value objects, and explicit SLF4J logger fields.

3. **Independent Maven projects per repo.** Each service and `snow-resorts-shared` has its own
   `pom.xml` and CI; shared libs are published to GitHub Packages and consumed by version pin
   (`snow-resorts-shared.version`), not via a sibling folder. Locally, `./mvnw install` in
   `snow-resorts-shared` populates `~/.m2` for offline service builds.

4. **One database, schema per service.** A single `snow_resorts` Postgres+PostGIS database with
   `auth` / `users` / `resorts` / `location` / `activity` schemas; each service owns its Flyway
   migrations. No cross-schema JOINs (communicate via API/events).

5. **Shared `security-lib` via Spring auto-configuration.** Adding the dependency wires the
   RFC 7807 handler, OWASP security-headers + correlation-id filters, and a default stateless
   JWT resource-server `SecurityFilterChain` (overridable per service via
   `@ConditionalOnMissingBean`). Package layout in the lib follows the same hexagonal
   vocabulary (`adapters` / `infrastructure` / `utils`).

6. **JWT: RS256 + JWKS.** `auth-service` issues RS256 tokens and publishes
   `/.well-known/jwks.json`; all other services validate as OAuth2 resource servers using that
   JWKS URI. Refresh tokens are opaque, single-use (rotation), stored only as SHA-256 hashes,
   with reuse-detection.

7. **Ports & Adapters (hexagonal package layout).** Every service uses `adapters` /
   `application` / `domain` / `infrastructure` / `utils`.
   - `domain`: `model`, pure rules, and `port` (outbound interfaces; no Spring / JPA / HTTP /
     Jakarta Validation).
   - `application`: `usecases` (`*UseCase` interfaces) and `service` (`*Service`
     implementations).
   - `adapters.inbound`: `controllers`, `dto` (HTTP Request/Response), `filters`, `realtime`
     (STOMP/WS).
   - `adapters.outbound`: `entities`, `repositories`, `storage`, `clients`, messaging, jwt,
     notification, redis, etc.
   - `infrastructure`: framework `config` (CORS/`SecurityConfig`/properties); shared API
     exceptions live under `infrastructure.config.exceptions`.
   - `utils`: mappers and non-domain helpers.
   - Dependency direction: `adapters` → `application` → `domain`. HTTP DTOs never sit in
     `domain`.
   - `contracts` keeps integration events (`com.snowresorts.contracts.events`).
   - Object storage uses an `ObjectStorage` port with an S3-compatible adapter pointed at
     MinIO (`local`) or S3 (`aws`); cache/pub-sub uses Redis.

## Consequences

- Local dev is $0 (Docker Compose + JVM). AWS only for **prod** in **`sa-east-1`** (owned by the
  Terraform workstream in `snow-resorts-infra`). Integration tests use Testcontainers
  (Postgres/PostGIS, Redis) and run in CI.
- Living platform docs live in the workspace root: [ARCHITECTURE.md](../../ARCHITECTURE.md),
  [LOCAL_DEV.md](../../LOCAL_DEV.md), and this `docs/adr/` tree.
