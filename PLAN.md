# PLAN.md

Every phase follows the same end-of-phase checklist (see `CLAUDE.md`):
- tests green and app run
- manual verification steps given
- boxes ticked, ADRs added
- conventional commit, push, **CI green**
- stop and wait for "go"

## Phase 0: Scaffold
- [ ] Gradle (Kotlin DSL) multi-project with committed wrapper, `buildSrc` conventions plugin and `gradle/libs.versions.toml`
- [ ] Spotless (google-java-format) + Checkstyle wired into `./gradlew check`; JaCoCo report generated
- [ ] `libs/common` module
- [ ] `api-service` and `runner-service` Spring Boot 3.5 skeletons with virtual threads enabled and `/actuator/health`
- [ ] `services/compactor` Node skeleton with `GET /health` and a test
- [ ] `web` Vite + React + TS skeleton with ESLint and `typecheck` script
- [ ] `compose.yaml` (Postgres 16, Redis 7, compactor) with healthchecks and memory limits
- [ ] `Makefile` (`up`, `dev`, `test`, `down`)
- [ ] `.github/workflows/ci.yml`:
  - [ ] Gradle check, JaCoCo report uploaded as an artifact
  - [ ] web lint + typecheck + build
  - [ ] compactor test
- [ ] `.devcontainer/devcontainer.json` updated (ports, postCreate warmup) and documented
- [ ] `.gitignore`, `.editorconfig`, README skeleton
- [ ] ADRs for any new decisions

## Phase 1: Auth and rooms
- [ ] Flyway V1: `users`, `refresh_tokens`, `rooms`, `room_members`, `invites`. PK/unique/FK constraints only.
- [ ] Register / login with BCrypt; `GET /api/me`
- [ ] Access JWT (HS256, 15 min) via oauth2-resource-server
- [ ] Refresh token:
  - [ ] opaque, hashed in DB, 7-day httpOnly + Secure + SameSite=Strict cookie on `/api/auth`
  - [ ] rotation on every refresh
  - [ ] family revocation on reuse
- [ ] Logout revokes the refresh family
- [ ] Rooms: create, list mine, get, rename, delete (owner only)
- [ ] Membership: list members, change role, remove member (owner only); roles OWNER / EDITOR / VIEWER
- [ ] Invite links: create (role, expiry, max uses), accept, revoke; token stored hashed
- [ ] Non-member → 404, insufficient role → 403
- [ ] Bean Validation on all requests; global `ProblemDetail` error handler
- [ ] springdoc OpenAPI + Swagger UI; every endpoint documented
- [ ] Unit tests (Mockito) for services; `*IT` with Testcontainers Postgres for each endpoint
- [ ] JaCoCo 80% gate on `..service..` packages enforced in CI

## Phase 2: Sandboxed execution
- [ ] Language images (Python, Node, Java, C++):
  - [ ] multi-arch, pinned base digests, non-root uid 10001
  - [ ] shared framed-stdin entrypoint that reports peak memory
- [ ] `docker-bake.hcl` + CI job that builds the images
- [ ] `SandboxExecutor` (docker-java) with:
  - [ ] `network=none`, read-only rootfs, size-capped `/workspace` tmpfs
  - [ ] memory = memory+swap, nanoCPUs, pids limit
  - [ ] `CapDrop ALL`, `no-new-privileges`, default seccomp, ulimits
  - [ ] wall-clock timeout, stdout/stderr byte cap with truncation flag
- [ ] Optional `runsc` runtime when configured and installed
- [ ] stdin support; result carries exit code, duration, peak memory
- [ ] Flyway V2: `runs` table; states QUEUED → RUNNING → COMPLETED / FAILED / TIMED_OUT, with monotonic transitions
- [ ] `POST /api/runs` → 202 `{runId}`, only room OWNER/EDITOR (viewer → 403, non-member → 404)
- [ ] `GET /api/runs/{id}`, room members only (others → 404)
- [ ] Authorization `*IT` tests for both endpoints (owner, editor, viewer, non-member)
- [ ] Minimal runner pipeline (`runs:jobs` → execute → `runs:events` → api persists). Results also pushed over WebSocket (minimal run-event socket, merged into the collaboration socket in Phase 4).
- [ ] Isolation tests against real Docker:
  - [ ] infinite loop → TIMED_OUT
  - [ ] fork bomb contained
  - [ ] network call fails
  - [ ] huge output truncated
  - [ ] write outside `/workspace` fails
  - [ ] gVisor test skips cleanly without `runsc`
- [ ] ADR: Docker socket risk and mitigations (ask about docker-socket-proxy before adding it)

## Phase 3: Queue and workers
- [ ] Consumer groups for `runs:jobs` (runners) and `runs:events` (api); groups created on startup
- [ ] Configurable worker concurrency (virtual threads + semaphore)
- [ ] `XACK` only after the result event is published
- [ ] Retry infrastructure failures with exponential backoff (ZSET `runs:retry`); after max attempts → `runs:dlq` + FAILED
- [ ] `XAUTOCLAIM` recovery of entries left pending by crashed workers
- [ ] `Idempotency-Key` header on `POST /api/runs`: unique `(user_id, idempotency_key)`, same key returns the same runId
- [ ] Graceful shutdown: stop reading, drain in-flight jobs within a timeout (`SmartLifecycle`)
- [ ] Per-user token-bucket rate limit (Redis Lua) → 429 + `Retry-After`
- [ ] Backpressure: queue lag over a threshold → 429
- [ ] `*IT` tests with Testcontainers Redis for each of the above, including a two-worker test and a kill-mid-job test

## Phase 4: Real-time collaboration
- [ ] `POST /api/rooms/{id}/ws-ticket` (30 s, single use, stored in Redis)
- [ ] Binary WebSocket `/ws/rooms/{id}?ticket=`, y-websocket wire format (sync, awareness, app=3 for run events)
- [ ] Flyway V3: `doc_updates` (append-only) and `doc_snapshots`
- [ ] Late joiner receives snapshot + tail
- [ ] Editor updates persisted and relayed; viewer updates dropped (awareness still relayed)
- [ ] Redis pub/sub fan-out so clients on different api instances share a room
- [ ] Compaction: past a count or byte threshold, take a Redis lock, call the compactor, store the snapshot, and delete compacted updates in one transaction
- [ ] Compactor: apply updates to `Y.Doc`, return `encodeStateAsUpdate`; rejects malformed input; tests
- [ ] Clean reconnect handling (session cleanup, awareness removal on disconnect)
- [ ] `*IT` tests:
  - [ ] two api instances share a room
  - [ ] late join
  - [ ] viewer edit rejected
  - [ ] compaction preserves document content
  - [ ] reconnect

## Phase 5: Frontend
- [ ] Auth flow: login/register, in-memory access token, silent refresh via cookie
- [ ] Room list + create room
- [ ] Editor page:
  - [ ] Monaco + y-monaco + custom y-protocols provider (reconnect with backoff)
  - [ ] live cursors with names and colors
- [ ] Language picker, run button, stdin input, output panel with live status (QUEUED/RUNNING/...)
- [ ] Share/invite link creation and accept page
- [ ] Viewer mode is read-only in the UI
- [ ] Vitest + RTL tests for key components and the provider
- [ ] One Playwright e2e flow (register → create room → type → run → see output; second browser sees the edit)
- [ ] Clean, minimal styling (ask before adding any CSS library)

## Phase 6: Performance
- [ ] Baseline run latency p50/p95 per language (cold containers), recorded in `docs/BENCHMARKS.md`
- [ ] Warm container pool per language (configurable size, replenished in the background, never reused across runs)
- [ ] p50/p95 before vs after, recorded
- [ ] k6: concurrent run submissions (throughput, error rate, p95)
- [ ] k6: concurrent editors per room (fan-out latency)
- [ ] Tune worker concurrency and Hikari pool from the data, within the 12 GB budget; update ADR-008
- [ ] Secondary indexes added only where `EXPLAIN ANALYZE` before/after shows a benefit, recorded

## Phase 7: Observability
- [ ] Micrometer + Prometheus registry; metrics: queue depth, run latency histogram per language, active rooms, active sockets, warm pool size
- [ ] Grafana dashboard JSON + provisioning committed under `deploy/grafana/`
- [ ] OpenTelemetry tracing: HTTP request → `runs:jobs` (`traceparent` field) → runner → `runs:events` (ask before adding a trace backend)
- [ ] Structured JSON logs with correlation IDs (request ID, runId, roomId)
- [ ] Prometheus with short retention and a memory limit

## Phase 8: Deploy and polish
- [ ] `deploy/compose.prod.yaml` with memory limits on every service and sandbox caps that fit the budget
- [ ] Caddyfile: automatic HTTPS for the DuckDNS subdomain, serves the React build, proxies `/api` and `/ws` (WebSocket upgrade verified)
- [ ] `.github/workflows/cd.yml`:
  - [ ] buildx multi-arch images → GHCR
  - [ ] deploy over SSH (secrets in Actions)
  - [ ] migrate step
  - [ ] health check, then automatic rollback to the previous tag
- [ ] Nightly Postgres backup to block storage with 7-day rotation
- [ ] `.env.example`; environment-based config; no secrets in the repo
- [ ] `docs/DEPLOY.md`:
  - [ ] Oracle A1 VM setup and Docker install
  - [ ] OCI security list + iptables rules
  - [ ] swap file
  - [ ] GHCR + secrets setup
- [ ] `docs/TUNNEL.md`: run locally and expose via Cloudflare Tunnel
- [ ] README:
  - [ ] Mermaid diagram, 3-command setup, Codespaces section
  - [ ] live demo link, demo GIF placeholder
  - [ ] benchmark table, security model
  - [ ] cost ("runs at $0/month"), trade-offs, future work
- [ ] End-to-end CD test once the VM and DuckDNS are ready
- [ ] Deliverables:
  - [ ] 4–5 resume bullets using only numbers from `docs/BENCHMARKS.md`
  - [ ] the 10 most likely interview questions with answers grounded in the code
