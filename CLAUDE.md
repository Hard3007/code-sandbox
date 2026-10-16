# CLAUDE.md

Real-time collaborative code sandbox (a mini Replit/CoderPad). It's a portfolio project, and it has to be production quality and cost $0 to run.
Phase checklist: `PLAN.md`. Design decisions: `docs/DECISIONS.md`. Measured numbers: `docs/BENCHMARKS.md`.

## Architecture summary

| Deployable | Tech | Role |
|---|---|---|
| `services/api-service` | Java 21, Spring Boot 3.5, virtual threads | Auth (JWT + refresh cookie), rooms/roles/invites, runs REST API, binary WebSocket collaboration gateway. **The only service with DB access** and the sole owner of Flyway migrations. |
| `services/runner-service` | Java 21, Spring Boot 3.5, docker-java | Consumes `runs:jobs` (Redis Streams consumer group `runners`), executes code in hardened throwaway containers (warm pool from Phase 6), publishes status to `runs:events`. Has no DB access. |
| `services/compactor` | Node 22, `yjs`, `node:http` | Stateless sidecar. api-service sends it a snapshot plus updates, and it merges them into a new snapshot. Internal network only. |
| `web` | React + TS + Vite, Monaco, Yjs, y-monaco | SPA. In production Caddy serves the build. |
| `sandbox-images/*` | Python, Node, Java, C++ | Pinned multi-arch language images with a shared framed-stdin entrypoint. |

Data flow:
- **Runs:** `POST /api/runs` → Postgres `runs` (QUEUED) + `XADD runs:jobs` → runner → `runs:events` → api-service persists the result, then publishes on `room:{id}` → every api instance pushes it to its sockets.
- **Collaboration:** Yjs updates over the y-websocket binary protocol → append-only `doc_updates` + Redis pub/sub `doc:{roomId}` → compaction into `doc_snapshots` through the compactor.
- **Infra:** Postgres 16, Redis 7, Caddy (HTTPS + WS proxy), Prometheus + Grafana, OpenTelemetry. Production is an Oracle Always Free ARM64 VM with a 12 GB memory budget.

## Working rules (non-negotiable)

1. **One phase at a time.** At the end of every phase:
   1. Run the full test suite (`make test`, the same checks CI runs).
   2. Run the app.
   3. Give the user exact manual verification steps.
   4. Tick the boxes in `PLAN.md`.
   5. Add the phase's ADRs.
   6. Make a conventional commit.
   7. `git push origin main` and watch CI (`gh run watch`).
   8. **A phase is not done until CI is green.** Then stop and wait for the user's "go".
2. **Never weaken, skip or delete a test to make it pass.** Fix the root cause.
3. **Record every key design decision** in `docs/DECISIONS.md` as an ADR (context, decision, trade-off).
4. Simple, readable code over clever code. **No leftover TODOs in finished phases.**
5. **Ask the user before adding any new major dependency.** The pre-approved list is in `docs/DECISIONS.md` (ADR-009).
6. **Zero cost is a hard requirement.** No paid services, nothing that can bill.
7. **Every image is multi-arch** (`linux/amd64` + `linux/arm64`), because production runs on ARM64.
8. **Production memory budget is 12 GB total.** Every service gets an explicit memory limit, and JVMs get a small `-Xmx`. Keep the budget table in ADR-008 up to date.
9. **Development happens only in GitHub Codespaces** via `.devcontainer/devcontainer.json`. Keep that file working, update it whenever a new tool is needed, and document it in the README.
10. **gVisor (`runsc`) is not available in Codespaces.** gVisor tests must skip cleanly with a JUnit assumption when `runsc` is missing. They must never fail or be deleted.

## Conventions

### Java
- Base package `dev.hard.sandbox` (`.api`, `.runner`, `.common`).
- Package by feature (`auth`, `room`, `run`, `collab`, ...). Inside each feature: `*Controller` → `*Service` → `*Repository`.
- Records for DTOs and events. Constructor injection only. No Lombok. No field injection.
- Errors are RFC 7807 `ProblemDetail` responses from one global `@RestControllerAdvice`. Throw domain exceptions; don't build responses in services.
- Validate requests with Bean Validation (`@Valid`). Document every public endpoint with springdoc annotations.
- Config goes in `@ConfigurationProperties` records. Secrets come from environment variables, never from committed files.
- Formatting: Spotless with google-java-format; Checkstyle config in `config/checkstyle/`.

### Database
- Flyway migrations in `services/api-service/src/main/resources/db/migration/V{n}__snake_case_description.sql`.
- Never edit an applied migration. Migrations must be backward compatible with the previous app version (expand/contract), because rollback only rolls back images.
- PK, unique and foreign key constraints (with explicit `ON DELETE`) go in from the start. Secondary indexes are added only in Phase 6, each justified with `EXPLAIN ANALYZE` in `docs/BENCHMARKS.md`.

### Authorization
- Room roles are `OWNER`, `EDITOR`, `VIEWER`.
- A non-member gets 404 on room-scoped resources, so resources can't be enumerated.
- A member without the needed role gets 403. For example, a viewer gets 403 on `POST /api/runs` and on sending doc edits.

### Tests
- JUnit 5. Unit tests (`*Test`) use Mockito for service logic.
- Integration tests (`*IT`) use **Testcontainers** Postgres and Redis, never mocks.
- Sandbox tests run against real Docker.
- JaCoCo gate: 80%+ line coverage on `..service..` packages, enforced by `./gradlew check`.
- Frontend: Vitest + React Testing Library. One Playwright e2e flow. Load tests in `load-tests/k6/`.

### Git
- Conventional commits (`feat:`, `fix:`, `test:`, `docs:`, `chore:`, `ci:`, `perf:`, `refactor:`), plus the attribution line the session requires.
- Work on `main`. Push after each phase.

## Commands

Each command becomes available in the phase that introduces it.

```bash
make up        # start dev dependencies (Postgres, Redis, compactor) via compose.yaml
make dev       # run api-service, runner-service and the Vite dev server
make test      # everything CI runs: ./gradlew check + web lint/typecheck/test + compactor test
make down      # stop dev dependencies

./gradlew check                              # Spotless check, Checkstyle, unit + integration tests, JaCoCo gate
./gradlew spotlessApply                      # auto-format Java
./gradlew :services:api-service:bootRun      # api-service on :8080 (Swagger UI at /swagger-ui.html)
./gradlew :services:runner-service:bootRun   # runner-service on :8081 (management only)

npm --prefix web run dev | lint | typecheck | test | build
npm --prefix web run e2e                     # Playwright e2e (needs the stack running)
npm --prefix services/compactor test

docker buildx bake -f sandbox-images/docker-bake.hcl   # build the language images
k6 run load-tests/k6/runs.js                           # load tests (Phase 6)
```

## Layout

```
.devcontainer/            Codespaces definition (the only supported dev environment)
buildSrc/                 Gradle convention plugin (toolchain, Spotless, Checkstyle, JaCoCo)
gradle/libs.versions.toml version catalog: all Java dependency versions live here
libs/common/              shared Java: stream/channel names, job and event records
services/api-service/     REST + WebSocket gateway + Flyway
services/runner-service/  job consumer + sandbox executor
services/compactor/       Yjs compaction sidecar (Node)
sandbox-images/           per-language runtime images + entrypoint
web/                      React client (+ e2e/)
compose.yaml              dev dependencies
deploy/                   production compose, Caddyfile, Prometheus, Grafana, scripts
load-tests/k6/            k6 scripts
docs/                     DECISIONS, BENCHMARKS, DEPLOY, TUNNEL
```
