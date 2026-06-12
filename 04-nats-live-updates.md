# Phase 4 — NATS: live updates, the CQRS split for real

**Goal:** commands stop returning HTML. Every page holds one long-lived SSE
connection; command handlers write Postgres, publish a tiny NATS event, return
`204`. The feed handler hears the event, re-queries, re-renders, pushes a
patch. Open two browsers — both update. This phase is the heart of the
template.

**Concepts:** NATS core pub/sub (fire-and-forget fan-out), subject taxonomy,
long-lived SSE streams, the command/query path split ("CQRS" here = separate
*routes*, not event sourcing — Postgres remains the read model).

---

## 1. Run NATS, learn the CLI

`nats-server` is already installed via mise (phase 0). Add tasks:

```toml
[tasks."dev:nats"]
run = "nats-server -js -sd ./.data/nats"

[tasks.dev]
depends = ["dev:nats", "dev:server"]
```

`mise run dev` now runs both in parallel — your Procfile. (`-js` enables
JetStream; unused until phase 6 but harmless now.)

Spend 10 minutes with the `nats` CLI — it's your debugger for everything
after this point:

```sh
nats sub 'events.>'           # leave running in a terminal
nats pub events.test 'hi'
```

## 2. Subjects: name them once

**Your task:** create `src/subjects.rs` with constants — never string-literal
a subject at a call site:

```rust
pub const MONITORS_CHANGED: &str = "events.monitors.changed";
```

Taxonomy to grow into: `events.>` for "re-render hints", `jobs.>` for the work
queue (phase 6). Keep event payloads **empty or tiny** (an id at most). The
event means "go look at Postgres", it does not carry state. This one decision
kills a whole class of consistency bugs.

## 3. Connect the client

```sh
cargo add async-nats
cargo add futures   # StreamExt for subscriptions
```

**Your task:** connect in `main` (`async_nats::connect(&nats_url)`), add the
`Client` to `AppState`. Like PgPool it's cheaply cloneable.

## 4. Flip the commands

**Your task:** rewrite both monitor commands to:

1. write Postgres,
2. `state.nats.publish(subjects::MONITORS_CHANGED, "".into()).await?`,
3. return `StatusCode::NO_CONTENT`.

Delete the fragment-rendering from command handlers. They no longer know the
UI exists. (Form-error patches on *invalid* input stay — validation feedback
is a direct response, not a state change.)

## 5. The feed

**Your task:** implement `GET /feeds/monitors` returning a **long-lived** SSE
stream that:

1. immediately yields a `PatchElements` of the current list fragment
   (so the feed also handles initial paint — the page template can render an
   empty `<div id="monitor-list">` shell),
2. subscribes to `events.monitors.>`,
3. on every message: re-query, re-render `fragments/monitor_list.html`,
   yield another `PatchElements`.

Wire it in the page: `<div data-on-load="@get('/feeds/monitors')">`.

Hints, because this is the genuinely new bit:

- `cargo add async-stream` — the `stream!`/`try_stream!` macro is the sane way
  to write this. The datastar SDK's axum `Sse` type wraps a
  `Stream` of Datastar events.
- The subscription is an async `Stream` of messages — `while let
  Some(msg) = sub.next().await` inside the `stream!` block.
- **Disconnects are normal.** When the browser tab closes, the stream is
  dropped; the subscription drops with it. Verify this: open/close tabs while
  watching `nats server report connections` or just log on stream creation.

Test the magic: two browser windows side by side. Create a monitor in one.
Also try `nats pub events.monitors.changed ''` from the CLI — UI updates
without any command. Sit with how weird and good that is: any process (the
worker in phase 6!) can now drive every connected user's UI.

## 6. Re-render scope — a design note to internalize

Right now any monitor change re-renders the whole list for every viewer.
That's the **correct default**: fragments are cheap, Postgres queries are
indexed, and coarse-grained re-rendering is simple to reason about. Granular
subjects (`events.monitors.{id}.changed`) and per-row fragments are an
optimization you apply *when a page measurably needs it* (phase 10 does this
for check ticks). Coarse first. Resist cleverness.

## 7. Commit

`jj describe -m "phase 4: nats pub/sub, sse feeds, commands return 204"` · `jj new`

---

## Checkpoints

- [ ] Network tab: exactly one `/feeds/monitors` request per page, status
      pending forever; commands are 204s with empty bodies.
- [ ] Two browsers stay in sync for create and delete.
- [ ] `nats pub events.monitors.changed ''` updates all browsers.
- [ ] Killing `nats-server` doesn't crash your server process (log + degrade;
      reconnect behavior of async-nats — read its docs on what happens by
      default and decide if it's acceptable for the template).
- [ ] You can explain why the event payload is empty.

## Stretch goals

- Heartbeat: yield a comment/keepalive event every ~20s so proxies don't kill
  idle SSE connections. The datastar SDK or axum's `Sse` has keepalive
  support — find it. Production config in phase 11 assumes this exists.
- Add `tracing` spans per feed connection: log connect/disconnect with a
  connection id. Observability DNA, cheap now.

Next: `05-workspace-refactor.md` — `main.rs` is bursting; give it a skeleton.
