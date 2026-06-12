# Phase 1 — Hello, Axum

**Goal:** a single-crate binary serving `GET /` with an HTML string, request
logging via tracing, and a hot-reload dev task. Smallest possible vertical
slice.

**Concepts:** tokio runtime, axum `Router`, handlers as plain async functions,
`tracing` + tower-http's `TraceLayer` for request logs.

---

## 1. Scaffold the crate

In the `magnetar` repo root:

```sh
cargo init --name magnetar
cargo add tokio --features full
cargo add axum
cargo add tracing tracing-subscriber --features tracing-subscriber/env-filter
cargo add tower-http --features trace
```

(Yes, single crate, code in `src/main.rs`. The workspace comes in phase 5,
*after* it hurts.)

## 2. The server

**Your task:** write `main.rs` so that:

1. `tracing_subscriber` initializes from the `RUST_LOG` env var
   (*Hint:* `tracing_subscriber::fmt().with_env_filter(EnvFilter::from_default_env())`).
2. An axum `Router` has routes: `GET /` → handler returning
   `Html("<h1>magnetar online</h1>")`, and `GET /healthz` →
   `StatusCode::OK` (every deployment in phase 11 wants it; cheap now).
3. A `TraceLayer::new_for_http()` layer on the router (see §3).
4. The port comes from a `PORT` env var, parsed to `u16`, defaulting to 3000.
5. It binds `0.0.0.0:{port}` with `tokio::net::TcpListener` and `axum::serve`,
   with `.expect("...")` messages that would actually help at 11pm — not bare
   `unwrap()`.
6. It logs one `tracing::info!` line with the bound address on startup.

Shape of the whole thing — fill in the blanks:

```rust
#[tokio::main]
async fn main() {
    // init tracing

    let app = axum::Router::new()
        .route("/", get(home))
        .route("/healthz", get(healthz))
        .layer(/* TraceLayer */);

    let port: u16 = std::env::var("PORT")
        .ok()                            // Result<String,_> -> Option<String>
        .and_then(|p| p.parse().ok())    // Option<String>   -> Option<u16>
        .unwrap_or(3000);

    // bind listener, info! the addr, axum::serve
}
```

Note the `.ok()` chain: `env::var` returns `Result<String, VarError>`;
`.ok()` converts to `Option` so fallible steps chain with `and_then`. A
plain `unwrap_or_else(|_| "3000".into())` on the Result also works for
strings — but parsing to `u16` means `PORT=garbage` can't produce a confusing
bind error. Decide your stance on garbage input — silently default, or
`.expect("PORT must be a number")` and die loudly — and keep it; phase 11
formalizes fail-fast config.

Things to actually understand before moving on (don't skip — this is the
foundation everything sits on):

- **Why `#[tokio::main]`?** What does it expand to? (*Hint:* `cargo expand`,
  or just read the macro docs.)
- **What makes a function a valid axum handler?** Skim the `axum::handler`
  docs — handlers are async fns whose params implement `FromRequestParts`/
  `FromRequest` and whose return implements `IntoResponse`. This one sentence
  explains 80% of axum.

## 3. Request logging

Bare axum/hyper log almost nothing by design. `TraceLayer` is the idiomatic
request-logging story: it creates a span per request (`method`, `uri`) and
emits an event per response (`status`, `latency`), all at **DEBUG** level
under the `tower_http` target — so `info` stays quiet, and you opt into
request logs with `RUST_LOG=info,tower_http=debug` in `.env.local`.

One tweak worth making immediately: the default latency unit is milliseconds,
which reads `0 ms` for every local request. Switch to microseconds:

```rust
use tower_http::{LatencyUnit, trace::{DefaultOnResponse, TraceLayer}};

.layer(
    TraceLayer::new_for_http()
        .on_response(DefaultOnResponse::new().latency_unit(LatencyUnit::Micros)),
)
```

(`DefaultOnResponse::new()` keeps the DEBUG level — we're only changing the
unit.)

Remember from phase 0: `RUST_LOG=debug mise run ...` is overridden by `.env` —
log-level changes go in `.env.local`.

## 4. Dev loop

Add to `mise.toml`:

```toml
[tasks."dev:server"]
run = "watchexec -r -e rs -- cargo run"
```

`-r` restarts the process on change (phase 2 adds `,html` to `-e` so template
edits rebuild too). `mise run dev:server`, edit the HTML string, watch it
restart. If you want a live error panel while coding, run `bacon clippy` in a
second terminal — that's bacon's job, not the task runner's (see phase 0).

## 5. Commit

```sh
jj describe -m "phase 1: hello axum"
jj new
```

---

## Checkpoints

- [ ] `curl localhost:3000` returns your HTML; `curl -i localhost:3000/healthz`
      returns 200.
- [ ] Request logs (with `method`, `uri`, `status`, latency in µs) appear
      when `tower_http=debug` is set in `.env.local`.
- [ ] Setting `RUST_LOG=debug` in `.env.local` is noisier (spans, hyper
      internals) than the default.
- [ ] You can explain in one sentence why `home` doesn't need any macro or
      registration beyond `.route()`.
- [ ] Editing the handler restarts the server automatically.
- [ ] `PORT=8080` in `.env.local` moves the server; garbage `PORT` behaves
      the way you *decided* it should.

## Stretch goals

- Bind address: `0.0.0.0` works everywhere including Docker (phase 11), but
  exposes the dev server to your LAN. If that bothers you, make `HOST` an env
  var with a `127.0.0.1` default in `.env` — a preview of the `Config` struct
  in phase 5.

Next: `02-templates-and-datastar.md` — real HTML and the first interactive bit.
