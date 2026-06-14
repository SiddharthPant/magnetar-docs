# Phase 2 — Askama templates + Datastar signals

**Goal:** the hello page becomes a real layout rendered by Askama, serving
static assets, with a Datastar-powered counter: a button POSTs to the server,
the server responds with an SSE patch, the DOM updates. No page reloads, no
JSON API, no client framework.

**Concepts:** compile-time templates, the Datastar mental model (signals on
the client, `PatchElements`/`PatchSignals` from the server), single-shot SSE
responses.

> **Read first (15 min):** the Datastar guide at data-star.dev — at minimum
> "Getting started" and the reference for `data-signals`, `data-on`,
> `data-text`, and backend actions (`@get`/`@post`). The whole framework is
> ~15 attributes; the time invested here pays off in every later phase.

---

## 1. Askama

```sh
cargo add askama
```

Create `templates/` at the crate root with:

- `layouts/base.html` — the HTML skeleton: head, the Datastar script tag, a
  `{% block content %}{% endblock %}`.
- `pages/home.html` — `{% extends "layouts/base.html" %}`, your content block.

**Your task:** define the template struct and render it:

```rust
#[derive(Template)]
#[template(path = "pages/home.html")]
struct HomePage { /* fields become template vars */ }
```

⚠️ **API churn warning:** older tutorials use the `askama_axum` glue crate —
it's gone in current askama. The pattern now is: call `.render()` yourself
(returns `Result<String>`), wrap in `axum::response::Html`. Write a tiny
helper now, because *every* page and fragment will use it:

```rust
fn render<T: Template>(t: &T) -> Result<Html<String>, AppError> { /* ... */ }
```

`AppError` doesn't exist yet — make it. **Your task:** create a minimal app
error type: a newtype around `anyhow::Error` (add `anyhow`) implementing
`IntoResponse` (log it, return 500). This is the error type all handlers will
return `Result<_, AppError>` with. *Hint:* the axum repo has an
`anyhow-error-response` example; understand it, don't copy it blind.

Askama renders at **compile time** — a typo in a template variable is a build
error. Break one on purpose and read the error so you recognize it later.

## 2. Static assets + Datastar

```sh
cargo add tower-http --features fs
mkdir -p assets && curl -L -o assets/datastar.js \
  https://cdn.jsdelivr.net/gh/starfederation/datastar@latest/bundles/datastar.js
```

(Vendor it — templates shouldn't depend on a CDN. Check data-star.dev for the
current bundle URL/version.)

**Your task:** serve `assets/` at `/assets` via `tower_http::services::ServeDir`
nested into the router, and load the script in `base.html` with
`type="module"`.

## 3. Dev loop update

Askama templates compile **into the binary** — editing a `.html` file must
trigger a rebuild. Update the watchexec task:

```toml
[tasks."dev:server"]
run = "watchexec -r -e rs,html -- cargo run"
```

## 4. The counter — your first command/patch cycle

This is the miniature of the whole architecture. In `pages/home.html`:

```html
<div data-signals="{count: 0}">
  <span id="count" data-text="$count"></span>
  <button data-on-click="@post('/commands/increment')">+1</button>
</div>
```

Server side:

```sh
cargo add datastar --features axum   # official Rust SDK; check docs.rs/datastar
cargo add serde --features derive
```

**Your task:** implement `POST /commands/increment`:

1. Extract signals with the SDK's `ReadSignals<T>` extractor, where
   `T` is `#[derive(Deserialize)] struct CounterSignals { count: i64 }`.
2. Respond with a Datastar SSE response containing a single
   `PatchSignals` event that sets `count` to the incremented value.
   *Hint:* the SDK's axum integration gives you an `Sse`/response type you can
   build from one event for one-shot responses like this — find it on docs.rs.

Now make a second button that increments via **`PatchElements`** instead:
render a fragment `<span id="count">…</span>` (Askama template
`fragments/count.html`) and send it; Datastar morphs it into the DOM by `id`.

**Stop and think — you just used both update channels:**

- `PatchSignals` → update client *state*, let bindings re-render.
- `PatchElements` → replace server-rendered *HTML* by id.

The template will use `PatchElements` for almost everything (HTML is the
engine of state — this is the hypermedia bet), and signals for ephemeral UI
state (open menus, form fields, toggles).

## 5. Commit

```sh
jj describe -m "phase 2: askama + datastar counter"
jj new
```

---

## Checkpoints

- [x] View-source on `/` shows server-rendered HTML; the counter works with
      the network tab showing a POST answered by `text/event-stream`.
- [x] Both variants (signals patch, elements patch) work and you can say when
      you'd use which.
- [x] A misspelled variable in a template fails `cargo build`, not runtime.
- [x] Your `render()` helper + `AppError` are in place — handlers return
      `Result<impl IntoResponse, AppError>`.

## Stretch goals

- Add a `data-indicator` to the button and disable it while the request is in
  flight — one attribute, big UX win, worth having in the template's DNA.
- Sketch (on paper) what `count` living in Postgres would change. That's
  phase 3.

## Implementation notes

- `AppError` is a newtype around `anyhow::Error` implementing `IntoResponse`. It
  does **not** implement `std::error::Error`, so Datastar SSE streams use
  `std::convert::Infallible` as their error type.
- The `+1 (elements)` button renders a fragment that still contains
  `data-text="$count"`. Datastar will overwrite the server-rendered number with
  the signal value after the element is patched, so the handler emits both
  `PatchSignals` (update the signal first) and `PatchElements` (replace the
  element second). This is a hybrid demo; in production, `PatchElements` is the
  primary channel for server-rendered HTML, while signals stay reserved for
  ephemeral UI state.
- `data-indicator:fetching` and `data-attr:disabled="$fetching"` were added to
  both buttons for the stretch-goal UX.

Next: `03-postgres-sqlx.md`.
