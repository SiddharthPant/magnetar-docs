# Magnetar — Build It Yourself

A phase-by-phase course for building **magnetar**: a Rust fullstack template
(Axum + Askama + SQLx + Postgres + Datastar + NATS) shaped as a small uptime
monitor with teams, roles, live dashboards, background workers, and a scheduler.

This is a **tutorial, not a codebase**. Each phase tells you *what* to build,
*why* it's shaped that way, and gives you hints and checkpoints — but you write
the code. By the end you'll have the `magnetar` template repo you'll clone for
every future client project, and more importantly you'll understand every line
in it.

## The destination

Three processes, one workspace:

```
server     axum: command routes (POST → 204) + SSE feeds (Datastar patches)
worker     NATS JetStream pull consumer: runs HTTP checks, opens incidents
scheduler  cron ticks: enqueues due checks, nightly cleanup, weekly digest
```

Postgres is the source of truth. NATS core pub/sub broadcasts "something
changed" to SSE feeds; JetStream is the durable job queue. The browser holds
one SSE connection per page and the server pushes re-rendered Askama fragments
through it. Commands never return HTML.

## Course map

| Phase | File | You build | New concept |
|-------|------|-----------|-------------|
| 0 | `00-tooling.md` | repo, mise, jj | toolchain as code |
| 1 | `01-hello-axum.md` | one route, one binary | axum router, tokio |
| 2 | `02-templates-and-datastar.md` | counter page | askama, signals, SSE patches |
| 3 | `03-postgres-sqlx.md` | monitors CRUD | migrations, query macros |
| 4 | `04-nats-live-updates.md` | multiplayer list | pub/sub → SSE feed |
| 5 | `05-workspace-refactor.md` | crates/ + apps/ | workspace hygiene |
| 6 | `06-worker-and-jobs.md` | "Check now" job | JetStream work queue |
| 7 | `07-scheduler.md` | auto checks | cron + dedup |
| 8 | `08-auth.md` | login, sessions | argon2, middleware |
| 9 | `09-teams-and-roles.md` | teams, invites, RBAC | authz extractors |
| 10 | `10-incidents-and-polish.md` | incidents, sparkline, toasts | fragments as projections |
| 11 | `11-production.md` | release build, deploy notes | sqlx offline, ops |

Phases 0–4 are deliberately done in a **single crate** with everything in
`main.rs` growing messy. That's intentional — phase 5 refactors into the real
workspace, and you'll *feel* why each crate boundary exists instead of
cargo-culting a structure.

## How to use this course

- Work in `~/projects/playground/magnetar` (the docs stay here, the code lives
  there). Phase 0 creates it.
- Every phase ends with **Checkpoints** — observable behaviors, not "it
  compiles". Don't move on until they all pass.
- Every phase ends with a `jj` commit. One phase ≈ one change. You'll have a
  clean history of the template growing.
- **Stretch goals** are optional. Skip them on first pass; they're noted so the
  template can grow later.
- When stuck for more than ~30 min on plumbing (not on understanding), it's
  fine to read a reference implementation — but type it out yourself, don't
  paste.

## Conventions used in the docs

- `Your task:` — the thing you implement.
- `Hint:` — a nudge: an API name, a crate feature flag, a docs link.
- Code blocks are **shapes, not solutions**: signatures, Cargo snippets,
  one-liners where the API is genuinely non-obvious. If a block looks complete,
  it's setup/config, not your homework.
- Crate versions are indicative (written June 2026). Check crates.io / the
  crate's docs.rs before pinning; the docs call out where APIs have churned
  recently (askama, datastar SDK).

## Prerequisites

You: comfortable in a typed language, basic SQL, basic HTTP. Rust basics help
but phases 1–3 are gentle; keep [The Book](https://doc.rust-lang.org/book/)
and [docs.rs](https://docs.rs) open. Machine: macOS with Docker (for
Postgres), and [mise](https://mise.jdx.dev) — phase 0 handles the rest.

Start with `00-tooling.md`.
