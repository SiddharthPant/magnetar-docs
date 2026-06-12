# Phase 1 — Hello, Axum

**Goal:** a single-crate binary serving `GET /` with an HTML string, plus
tracing logs and a hot-reload dev task. Smallest possible vertical slice.

**Concepts:** tokio runtime, axum `Router`, handlers as plain async functions,
`tracing` for logs.

---

## 1. Scaffold the crate

In the `magnetar` repo root:

```sh
cargo init --name magnetar
cargo add tokio --features full
cargo add axum
cargo add tracing tracing-subscriber --features tracing-subscriber/env-filter
```

(Yes, single crate, code in `src/main.rs`. The workspace comes in phase 5,
*after* it hurts.)

## 2. The server

**Your task:** write `main.rs` so that:

1. `tracing_subscriber` initializes from the `RUST_LOG` env var
   (*Hint:* `tracing_subscriber::fmt().with_env_filter(EnvFilter::from_default_env())`).
2. An axum `Router` has one route: `GET /` → handler returning
   `Html("<h1>magnetar online</h1>")`.
3. It binds `0.0.0.0:3000` with `tokio::net::TcpListener` and `axum::serve`.
4. It logs one `tracing::info!` line with the bound address on startup.

Shape of the whole thing — fill in the blanks:

```rust
#[tokio::main]
async fn main() {
    // init tracing

    let app = axum::Router::new().route("/", get(home));

    // bind listener, info! the addr, axum::serve
}

async fn home() -> Html<&'static str> { /* ... */ }
```

Things to actually understand before moving on (don't skip — this is the
foundation everything sits on):

- **Why `#[tokio::main]`?** What does it expand to? (*Hint:* `cargo expand`,
  or just read the macro docs.)
- **What makes a function a valid axum handler?** Skim the `axum::handler`
  docs — handlers are async fns whose params implement `FromRequestParts`/
  `FromRequest` and whose return implements `IntoResponse`. This one sentence
  explains 80% of axum.

## 3. Dev loop

Add to `.mise.toml`:

```toml
[tasks."dev:server"]
run = "bacon run"          # bacon re-runs on file change; `bacon` alone = check loop
```

`mise run dev:server`, then edit the HTML string and watch it restart.

## 4. Commit

```sh
jj describe -m "phase 1: hello axum"
jj new
```

---

## Checkpoints

- [ ] `curl localhost:3000` returns your HTML.
- [ ] `RUST_LOG=debug mise run dev:server` is noisier than `info`.
- [ ] You can explain in one sentence why `home` doesn't need any macro or
      registration beyond `.route()`.
- [ ] Editing the handler restarts the server automatically.

## Stretch goals

- Add a `/healthz` route returning `StatusCode::OK` — every deployment in
  phase 11 will want it. Cheap now.
- Make the port come from a `PORT` env var with a default. (*Hint:*
  `std::env::var("PORT").ok()` — resist pulling in a config crate; the
  template's config story is env vars, full stop.)

Next: `02-templates-and-datastar.md` — real HTML and the first interactive bit.
