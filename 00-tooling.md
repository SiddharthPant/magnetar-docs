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

## 2. Declare your tools in `.mise.toml`

Create `.mise.toml` at the repo root:

```toml
[tools]
rust = "1.88"                          # check rust-lang.org for current stable
"cargo:sqlx-cli" = "latest"
"cargo:bacon" = "latest"
"ubi:nats-io/nats-server" = "latest"
"ubi:nats-io/natscli" = "latest"       # the `nats` debug CLI
watchexec = "latest"

[env]
_.file = ".env"

[tasks.hello]
run = "echo magnetar online"
```

Then:

```sh
mise trust && mise install
mise run hello
```

Notes worth internalizing:

- `[env] _.file = ".env"` means **you will not use the dotenvy crate**.
  Anything run via mise (tasks, `mise exec`, your shell if activated) gets
  `.env` injected. In production the env comes from systemd/containers, so the
  binaries only ever read real environment variables. One less crate, one less
  codepath.
- mise replaces rustup here. `cargo`, `rustc` etc. are shimmed to the pinned
  version per-project.
- We install `nats-server` as a dev tool because it's a single static binary —
  Postgres stays in Docker (phase 3) because databases with state don't belong
  in a tool manager.

## 3. `.env`, `.env.example`, `.gitignore`

**Your task:** create three files.

`.env.example` (committed — the contract):

```sh
DATABASE_URL=postgres://magnetar:magnetar@localhost:5432/magnetar
NATS_URL=nats://localhost:4222
RUST_LOG=info,sqlx=warn
```

`.env` — copy of the above, gitignored, yours to edit.

`.gitignore`:

```
/target
.env
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
- [ ] `rustc --version` inside the repo shows the pinned toolchain.
- [ ] `jj log` shows your described phase-0 change with a new empty `@` on top.
- [ ] `git log` shows the same commit (colocation working).

## Stretch goals

- Add `[settings] experimental_monorepo_root = true`? No — resist. Add nothing
  speculative; the template earns features per phase.
- Read `mise tasks --help` and note that `depends` runs in parallel. You'll use
  that in phase 4 to get a Procfile-style `mise run dev`.

Next: `01-hello-axum.md` — first Rust.
