# Architecture Decision Records

Short ADRs: context, decision, trade-off. Numbered in order and never renumbered. When a decision is replaced, mark the old one as superseded.

---

## ADR-001: Monorepo with a Gradle multi-project build

**Context.** There are three or more deployables: api-service, runner-service, web, plus the compactor sidecar. They share message contracts such as stream names and job/event records. The project has a single developer, and reviewers should be able to read it in one place.

**Decision.** Use one repository.
- The Java services and `libs/common` are subprojects of a single Gradle (Kotlin DSL) build.
- A `buildSrc` convention plugin holds the shared settings: toolchain, Spotless, Checkstyle, JaCoCo.
- All versions live in one `gradle/libs.versions.toml`.
- `web` and `services/compactor` are plain npm projects in the same repo.

**Trade-off.**
- Gains: atomic cross-service changes, one CI pipeline, shared contracts compiled together.
- Costs: CI builds everything on every change unless filtered by path, and the services can't be versioned independently. That's acceptable for a single-team project.

---

## ADR-002: GitHub Codespaces is the only supported dev environment

**Context.** The project needs Java 21, Node 22, Docker (for Testcontainers and the sandbox) and buildx. "Works on my machine" drift wastes time, and reviewers should be able to open the repo and run it.

**Decision.**
- `.devcontainer/devcontainer.json` defines the full toolchain, using Docker-in-Docker for container work.
- It's the only supported environment, kept working and updated whenever a tool is added.
- gVisor (`runsc`) can't run in Codespaces, so gVisor tests are guarded by a JUnit assumption and skip cleanly when it's missing.

**Trade-off.**
- Gains: reproducible setup in one click.
- Costs:
  - The dev box is x86_64 while production is ARM64. Multi-arch builds through buildx/QEMU cover this, but ARM-specific behaviour is only proven in CI and on the VM.
  - gVisor is only exercised in production.

---

## ADR-003: Yjs compaction in a stateless Node sidecar

**Context.**
- Documents are stored as an append-only Yjs update log, which grows without bound, and late joiners would replay all of it.
- Merging updates correctly needs the Yjs CRDT algorithm, and there's no mature Java implementation.
- Letting a client upload "the full state" would let one malicious editor overwrite the document.

**Decision.**
- `services/compactor` is a tiny Node service using the real `yjs` library and `node:http` (no framework).
- When a document's log passes a count or byte threshold, api-service takes a Redis lock and posts the current snapshot plus the pending updates.
- The compactor applies them to a fresh `Y.Doc` and returns `Y.encodeStateAsUpdate`, which also garbage-collects deleted content.
- In one transaction, api-service stores the new snapshot with the highest update id it covers and deletes the compacted updates.
- The compactor is stateless, reachable only on the internal Docker network, and never exposed.
- The server never accepts document state from clients.

**Trade-off.**
- Gains: correct merges with real GC.
- Costs: a fourth deployable, a second backend language, about 100 MB of RAM, and a network hop on a background path.
- If the compactor is down, compaction is just delayed. Editing and joining keep working from the snapshot plus a longer tail.

---

## ADR-004: runner-service has no database access; results flow through a Redis stream

**Context.** Runners scale horizontally, and every runner with a DB pool multiplies Postgres connections. Postgres has a small memory budget (ADR-008). Two services writing one schema also blurs ownership.

**Decision.**
1. Runners read jobs from `runs:jobs` and publish status changes (RUNNING, COMPLETED, FAILED, TIMED_OUT) to the `runs:events` stream.
2. api-service consumes `runs:events` in its own consumer group and persists each event idempotently. Transitions are monotonic: a terminal state is never overwritten.
3. It then fans the event out over Redis pub/sub to every api instance's WebSockets.
4. api-service is the sole owner of the schema and the Flyway migrations.

**Trade-off.**
- Gains: fewer DB connections, a smaller runner heap, clear ownership.
- Results are durable even if every api instance is down, because events wait in the stream.
- Costs: `GET /api/runs/{id}` is eventually consistent, typically a few milliseconds behind the runner. There's also one more stream to monitor.

---

## ADR-005: Refresh token in an httpOnly cookie, with rotation and reuse detection

**Context.** Access tokens must be short-lived. The SPA needs silent re-authentication, and any token readable from JavaScript can be stolen by XSS.

**Decision.**
- **Access token:** HS256 JWT, 15-minute TTL, validated by Spring's oauth2-resource-server and kept only in memory on the client.
- **Refresh token:**
  - opaque random value, 7-day TTL
  - stored as a SHA-256 hash in Postgres
  - sent as an `httpOnly; Secure; SameSite=Strict` cookie scoped to `/api/auth`
- **Rotation:** every refresh issues a new token in the same family and revokes the old one.
- **Reuse detection:** presenting a revoked token revokes the whole family.
- **Logout:** revokes the family.

**Trade-off.**
- Gains: XSS can't exfiltrate the refresh token, and stolen-token replay is detected.
- Costs:
  - Testing in curl or Swagger needs cookie handling.
  - Every refresh is a DB write.
  - HS256 means every validating service shares a secret. That's fine while only api-service validates; switch to RS256 if other services need to verify tokens.

---

## ADR-006: WebSocket authentication with short-lived single-use tickets

**Context.** Browsers can't set an `Authorization` header on a WebSocket handshake. Putting the JWT in the query string leaks it into proxy and access logs.

**Decision.**
1. The client calls `POST /api/rooms/{id}/ws-ticket` with its access token.
2. The server stores a random ticket in Redis (30 s TTL, bound to user and room) and returns it.
3. The client connects to `/ws/rooms/{id}?ticket=...`. The handshake interceptor consumes the ticket atomically with `GETDEL`, so it can only be used once.

**Trade-off.**
- Gains: no long-lived credential in URLs, and a leaked ticket is useless after 30 s or one use.
- Costs: one extra round trip per connect or reconnect, and Redis becomes a dependency of the handshake.

---

## ADR-007: Deliver source code to sandboxes over framed stdin

**Context.** Sandboxes use a read-only root filesystem with a tmpfs workdir. `docker cp` can't write into a tmpfs mount, and a read-only rootfs blocks it anyway. Environment variables and arguments have size limits and show up in `docker inspect`. The warm pool (Phase 6) needs containers that start before the code is known.

**Decision.**
1. The runner attaches to the container's stdin and writes a header line with the source length in bytes, then the source.
2. Everything after that is the user's stdin.
3. The image's entrypoint reads the header, copies exactly that many bytes into `/workspace` with `head -c`, then `exec`s the language command, which inherits the remaining stdin.

**Trade-off.**
- Gains: one mechanism works for cold and warm containers, with no writable rootfs and no size-limited environment variables.
- Costs: the entrypoint is a small piece of custom protocol that must be tested per image, and a single-file program model.

---

## ADR-008: Production memory budget (12 GB, Oracle A1). DRAFT, refined with Phase 6 data

**Context.** The production VM has 12 GB RAM and 2 ARM64 cores, and the hard limit is $0. Unbounded JVM heaps, Redis or Prometheus could trigger the OOM killer, and so could a burst of sandboxes.

**Decision.** Every container gets an explicit `mem_limit`. JVMs get a fixed `-Xmx` well below their limit to leave room for metaspace, thread stacks and direct buffers.

| Service | Limit | Settings |
|---|---|---|
| caddy (serves web build) | 64 MB | |
| api-service | 768 MB | `-Xmx384m` |
| runner-service | 512 MB | `-Xmx256m` |
| compactor | 128 MB | `--max-old-space-size=96` |
| postgres | 512 MB | `shared_buffers=128MB` |
| redis | 192 MB | `maxmemory 128mb`, `maxmemory-policy noeviction` |
| prometheus | 256 MB | `--storage.tsdb.retention.time=3d` |
| grafana | 192 MB | |
| sandboxes | ~3.3 GB | max 4 concurrent × ≤512 MB + warm pool of 1 per language |
| **Subtotal** | **~5.9 GB** | |
| OS + dockerd | ~2 GB | |
| Headroom | ~4 GB | second api or runner instance during deploys, compile spikes; plus a 4 GB swap file as a safety net |

- Redis uses `noeviction` because evicting stream entries would silently lose jobs. The API returns 429 through backpressure (Phase 3) before Redis fills up.

**Trade-off.**
- Gains: a fixed worst-case footprint that can't OOM the host.
- Costs: low concurrency (4 parallel runs) and small heaps. These limits are revisited with measured data in Phase 6 and recorded in `docs/BENCHMARKS.md`.

---

## ADR-009: Pre-approved dependencies

**Context.** The working rules require asking before adding any major dependency. These were approved up front so the plan could be set:
- **Backend:** Spring Security + `spring-boot-starter-oauth2-resource-server` (Nimbus JWT), springdoc-openapi, docker-java (httpclient5 transport).
- **Build and test:** Testcontainers, JaCoCo, Spotless (google-java-format), Checkstyle.
- **Frontend:** Vitest + React Testing Library, y-protocols, react-router.
- **Compactor:** `yjs`, `node:http` (no framework).

**Decision.** These can be added in the phase that needs them without asking again. Everything else requires approval first. Known upcoming asks:
- docker-socket-proxy (Phase 2)
- a trace backend such as Jaeger or Tempo (Phase 7)
- possibly a CSS approach (Phase 5)

**Trade-off.** Mainstream, well-documented libraries keep the code easy to defend in interviews, at the cost of some transitive weight. This matters most for Spring Security.
