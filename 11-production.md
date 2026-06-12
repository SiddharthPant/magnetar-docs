# Phase 11 — Production: ship it, then freeze the template

**Goal:** the workspace builds offline in CI, produces three small release
binaries (or one multi-call binary — you'll decide), migrations run on deploy,
SSE survives a reverse proxy, and the repo gets the final README that makes it
a *template*. No new features.

---

## 1. sqlx offline mode

CI has no database, but `query!` needs one. **Your task:**

```sh
cargo sqlx prepare --workspace
```

Commit the generated `.sqlx/` directory. Builds with `SQLX_OFFLINE=true` now
type-check against the committed metadata. Add a mise task `prepare`, and a
`check:ci` task: `SQLX_OFFLINE=true cargo clippy --workspace --all-targets
-- -D warnings` + `cargo test --workspace`. Rule for the README: **changing
any SQL ⇒ re-run `mise run prepare` in the same change.** (A CI step that
fails when `.sqlx` is stale: run prepare, `jj diff`/`git diff --exit-code`.)

## 2. Embedded migrations

Swap CLI-run migrations for self-migrating binaries: in `bootstrap()` (or
server-only — think about it: three replicas racing migrations? sqlx takes a
lock, but decide *which process* migrates and write it down; common answer:
server migrates, worker/scheduler `migrate!().validate`-or-wait):

```rust
sqlx::migrate!("../../migrations").run(&pool).await?;
```

(Path is relative to the crate — get it right for wherever bootstrap lives.)

## 3. Release builds

**Your task:** root `Cargo.toml`:

```toml
[profile.release]
lto = "thin"
codegen-units = 1
strip = true
```

Multi-stage `Dockerfile`: `rust:1.x` builder (with `SQLX_OFFLINE=true`),
`debian:bookworm-slim` (or distroless) runtime, copy the three binaries +
`assets/`. One image, three commands — compose/Dokploy services differ only
in `command:`. *Hint:* `cargo build --release --workspace` builds all three;
mount a cargo registry cache in CI.

**Decision point — one binary instead?** A single `magnetar` binary with a
`serve|work|schedule` subcommand (clap) simplifies images and version skew.
Tradeoff: can't update worker deps without rebuilding everything (you share a
workspace anyway — moot). Template recommendation: **multi-call binary**.
Refactor `apps/*` mains into `run()` fns called from one `apps/magnetar`
if you agree. Either way: README.

## 4. SSE behind a proxy

The classic deploy-day failure. Checklist for Traefik/nginx/Caddy in front:

- response buffering **off** for `/feeds/*` and `/t/*/feeds/*`
  (nginx: `proxy_buffering off` / `X-Accel-Buffering: no` header — you can
  set that header from axum on feed responses; do it, it's self-defending),
- read/idle timeouts > your keepalive interval (you did the phase-4 stretch
  heartbeat, right? if not, now),
- HTTP/1.1 minimum to upstream, no connection-level compression on streams.

**Your task:** set the `X-Accel-Buffering: no` header on all feed responses
in code, add the keepalive if missing, and write `docs/deploy.md` in the
repo capturing the proxy checklist for your usual stack (Dokploy/Traefik —
record actual labels/middleware you verified, not theory).

## 5. Config + ops final pass

- `Config` in core: fail-fast on missing env at boot with a *named* error
  per var (a client deploy at 11pm will hit this; make the message tell them
  the var name). List every var in `.env.example` — that file is the
  config documentation.
- Graceful shutdown for all three processes (phase-6 stretch becomes
  mandatory): axum `with_graceful_shutdown`, worker finishes in-flight,
  scheduler just exits.
- `/healthz` returns 200 only if DB and NATS are reachable (cheap `select 1`
  + `client.connection_state()`); used by compose healthchecks.
- Structured logs: `tracing_subscriber` json formatter behind an env flag
  (`LOG_FORMAT=json`).

## 6. Freeze it as a template

**Your task:** the final README of the *magnetar repo* (not these docs):

1. Pitch paragraph + architecture diagram (ASCII fine): three processes,
   Postgres truth, NATS seams.
2. Quickstart: `mise install && mise run db:up && mise run dev` — verify on a
   clean clone that this is literally true.
3. The recorded decisions you wrote along the way (sessions-not-JWT, path
   tenancy, coarse-then-granular rendering, msg-id dedup, multi-call binary,
   hand-rolled CSS…). This section is what makes it *your* template.
4. "Starting a client project from magnetar": rename checklist (crate names,
   strings, `JOBS` stream name if running shared NATS), what to rip out
   (monitors domain) vs keep (auth, teams, jobs, feeds plumbing).

Tag it: `jj bookmark set v0.1.0 -r @-` and push. Done.

---

## Checkpoints

- [ ] `SQLX_OFFLINE=true mise run check:ci` passes with Postgres **stopped**.
- [ ] `docker compose up` from the production compose runs all three
      processes + deps; fresh DB self-migrates; demo works through the proxy
      (SSE included) — this is the real graduation exercise.
- [ ] Kill -TERM each process under load: no half-done jobs, no dropped
      requests logged as errors.
- [ ] A colleague (or you, next month) can go clone → running in under 10
      minutes using only the README.

## Where the template deliberately stops

Billing, OAuth/SSO, real email delivery, notification channels, audit logs,
multi-region workers, charts libraries. Each is a client-project feature with
client-specific shape. The template's job was the *skeleton*: tenancy, authz,
durable jobs, scheduling, live hypermedia. That job is done.

बधाई हो — you didn't just build a template, you now know why every line of it
exists. That's the part no starter repo on GitHub could have given you.
