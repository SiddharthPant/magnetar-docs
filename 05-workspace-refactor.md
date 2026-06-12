# Phase 5 — Workspace refactor: the magnetar skeleton

**Goal:** split the single crate into the workspace shape every future client
project will start from. **No new features.** App behaves identically at the
end; `jj diff` shows only moves and plumbing.

**Concepts:** cargo workspaces, workspace-level dependency management, crate
boundaries as architecture, the shared `bootstrap()` that all binaries use.

---

## 1. Target shape

```
magnetar/
├─ Cargo.toml            # [workspace] + [workspace.dependencies]
├─ migrations/           # stays at root (sqlx-cli default)
├─ assets/
├─ compose.yaml
├─ crates/
│  ├─ core/              # Ctx + bootstrap(), config, ids, AppError, subjects
│  ├─ db/                # PgPool setup, Monitor model, all queries
│  ├─ bus/               # NATS connect, subject constants, (later: JetStream)
│  └─ web/               # askama templates + render fns + the render() helper
│     ├─ askama.toml
│     └─ templates/
└─ apps/
   └─ server/            # axum router, handlers, main.rs
```

(`worker/` and `scheduler/` arrive in phases 6–7. `jobs/` crate too.)

## 2. Mechanics

**Your task**, step by step — keep it compiling as often as possible:

1. Root `Cargo.toml` → `[workspace] members = ["crates/*", "apps/*"]`,
   `resolver = "2"`. Move **every** dependency version into
   `[workspace.dependencies]`; member crates then use
   `axum = { workspace = true }`. One place to bump versions — non-negotiable
   template hygiene.
2. `cargo new --lib crates/core` (and db, bus, web); `cargo new apps/server`.
   Move code outward-in: subjects + error → core, queries → db, templates +
   render fns → web, the router/handlers/main → apps/server.
3. Templates: askama looks for `templates/` relative to the crate using the
   derive — so they live in `crates/web/templates/`. Add
   `crates/web/askama.toml` if you want to configure dirs explicitly.
4. Dependency *direction* (enforce it in your head, cargo enforces cycles):

   ```
   core  ←  db, bus, web  ←  apps/*
   ```

   `web` may depend on db **types** (to render a `Monitor`) but never runs
   queries. Handlers (apps/server) fetch, then hand data to web to render.
   If you feel friction here, good — that friction is the design.

## 3. `Ctx` and `bootstrap()`

The payoff of this phase. In `core`:

```rust
#[derive(Clone)]
pub struct Ctx {
    pub db: sqlx::PgPool,
    pub nats: async_nats::Client,
    pub cfg: Config,          // your env-var struct: addrs, urls
}

pub async fn bootstrap() -> anyhow::Result<Ctx> { /* read env, connect both */ }
```

`apps/server/src/main.rs` shrinks to: `bootstrap()`, build router with
`Ctx` as state, serve. **Every future binary is `bootstrap()` + a loop.**
That's the whole multi-process story. (Coming from Effect: `Ctx` is your
composed `Layer`, `bootstrap` is the runtime construction — just without the
type-level dependency graph.)

> Wait — does `bootstrap` in core depend on db/bus crates, breaking the
> direction rule? Two clean options: (a) core defines `Ctx` over the raw pool/
> client types only (sqlx + async_nats are then core deps — fine, they're
> foundational), construction helpers live where they belong; (b) bootstrap
> lives in a thin `crates/app` glue crate. **Your task:** pick one,
> write a sentence in the root README justifying it. Templates need recorded
> decisions more than perfect ones.

## 4. Tasks touch-up

`mise.toml`: dev task now `watchexec -r -e rs,html -- cargo run -p server`.
Add `[tasks.check] run = "cargo clippy --workspace --all-targets -- -D warnings"`
and make it pass. Add `cargo fmt --check` to taste.

## 5. Commit

`jj describe -m "phase 5: workspace refactor — crates/{core,db,bus,web} + apps/server"` · `jj new`

This is also a great moment for jj practice: if you made a mess, `jj split`
the change into "move templates" / "move queries" / "workspace plumbing".

---

## Checkpoints

- [ ] App behaves exactly as at the end of phase 4 (two-browser test again).
- [ ] `cargo build -p server` works; `cargo clippy --workspace` is clean.
- [ ] No version numbers in member `Cargo.toml`s — all `workspace = true`.
- [ ] You can state the dependency direction from memory and name where
      `bootstrap` lives and why.

## Stretch goals

- `cargo deny` or at least `cargo tree -d` to spot duplicate dep versions.
- A `crates/web` unit test that renders one fragment with fixture data and
  asserts on the string — the cheapest UI regression net you'll ever buy.

Next: `06-worker-and-jobs.md` — the second process.
