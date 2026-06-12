# Phase 0 — Tooling: mise, jj, and an empty repo done right

**Goal:** an empty `magnetar` repo where the toolchain, env vars, and task
runner are all declared in one file, and version control is jj colocated with
git. Zero Rust code yet.

**Why this matters for a template:** when you clone magnetar for a client
project in 6 months, `mise install && mise run dev` must be the entire setup.
Everything in this phase is what makes that true.

---

## 1. Create the repo

```sh
mkdir -p ~/projects/playground/magnetar && cd $_
git init
jj git init --colocate
```

Colocated means `.git` and `.jj` live side by side: every jj command syncs to
git automatically, so GitHub/CI/editor tooling keep working, and you can always
fall back to raw `git`. jj adds `.jj/` to `.git/info/exclude` itself.

If you haven't configured jj yet:

```sh
jj config set --user user.name  "Siddharth"
jj config set --user user.email "you@example.com"
```

**Your task:** skim `jj help -k tutorial` (or the official jj tutorial) just
enough to understand three ideas: the working copy *is* a commit, `jj new`
starts the next change, `jj describe -m` names the current one. There is no
staging area. That's 90% of daily jj.

## 2. Declare your tools in `mise.toml`

Create `mise.toml` at the repo root (mise accepts `mise.toml` or `.mise.toml`;
we use the visible one):

```toml
[tools]
rust = "1.96.0"                          # pin current stable explicitly
"cargo:sqlx-cli" = "latest"
"cargo:bacon" = "latest"                 # interactive check loop — see note below
"github:nats-io/nats-server" = "latest"  # github backend (ubi is deprecated)
"github:nats-io/natscli" = "latest"      # the `nats` debug CLI
watchexec = "latest"                     # drives all dev:* restart tasks

[env]
_.file = [".env", ".env.local"]          # layered: later files win

[tasks.hello]
run = "echo magnetar online"
```

Then:

```sh
mise trust && mise install
mise run hello
```

Notes worth internalizing:

- **Env layering + the override gotcha.** `_.file` injects both files into
  anything mise runs; `.env.local` (gitignored) overrides `.env` (committed).
  Crucially, mise's `[env]` **overrides your shell environment** — so
  `RUST_LOG=debug mise run dev:server` will be stomped by the value in `.env`.
  To change log levels (or anything), edit `.env.local`, don't prefix the
  command. Verify the behavior once so you trust it:
  `RUST_LOG=debug mise exec -- env | grep RUST_LOG`.
- Because of this setup, **you will not use the dotenvy crate**. In
  production the env comes from systemd/containers; binaries only ever read
  real environment variables.
- **watchexec vs bacon — they overlap, with different jobs.** watchexec is a
  generic "kill and rerun on file change" tool with plain output: right for
  parallel `mise run dev` tasks. bacon is a Rust-specific TUI (parses compiler
  output, persistent error panel) that wants its own terminal — and its
  default `wait_then_restart` strategy never restarts a long-running server.
  Rule: **watchexec in mise tasks, bacon by hand** (`bacon clippy` next to
  your editor) when deep in a crate.
- mise replaces rustup here; `cargo`/`rustc` are shimmed to the pinned version
  per-project. We install `nats-server` as a dev tool because it's a single
  static binary — Postgres stays in Docker (phase 3) because databases with
  state don't belong in a tool manager.

## 3. Env files and `.gitignore` — the magnetar convention

This template **commits `.env`** and gitignores `.env.local`. That inverts
the common ".env is secret" convention, deliberately: the committed `.env` is
the *documented default config* (it replaces `.env.example` — no drift between
example and reality), and all secrets/overrides go in `.env.local`.

The rule that makes this safe — write it at the top of `.env` itself:

```sh
# Committed defaults — safe for any machine.
# NEVER put real secrets here. Secrets and overrides go in .env.local (gitignored).
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app_db
NATS_URL=nats://localhost:4222
RUST_LOG=info,sqlx=warn
PORT=3000
```

`.gitignore`:

```
.env.local
/target
/.data
```

`.data/` will hold the JetStream store and any local scratch state.

## 4. First change

```sh
jj describe -m "phase 0: tooling — mise, env contract, repo init"
jj new
```

`jj new` opens the next (empty) change; from now on you're always "inside" the
commit you're building. Run `jj log` and stare at it for a second — `@` is you.

---

## Checkpoints

- [ ] `mise run hello` prints from inside the repo.
- [ ] `mise exec -- env | grep DATABASE_URL` shows the value from `.env`.
- [ ] `RUST_LOG=debug mise exec -- env | grep RUST_LOG` shows `.env` winning
      over your shell — and you know where overrides actually go.
- [ ] `rustc --version` inside the repo shows the pinned toolchain.
- [ ] `jj log` shows your described phase-0 change with a new empty `@` on top.
- [ ] `git log` shows the same commit (colocation working).

## Stretch goals

- Read `mise tasks --help` and note that `depends` runs in parallel. You'll
  use that in phase 4 to get a Procfile-style `mise run dev`.
- Resist adding anything speculative; the template earns features per phase.

Next: `01-hello-axum.md` — first Rust.
