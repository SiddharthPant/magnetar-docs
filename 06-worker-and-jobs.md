# Phase 6 — Worker + JetStream: durable background jobs

**Goal:** a second binary. A "Check now" button on a monitor enqueues a job;
the **worker** process pulls it from a JetStream work-queue stream, performs a
real HTTP check, writes a `checks` row, publishes an event — and the dashboard
ticks live. Server never does the work; worker never talks to a browser.

**Concepts:** JetStream streams/consumers vs core NATS, work-queue retention,
ack/nak with backoff, at-least-once delivery → idempotency, the shared `Job`
enum, NATS as the seam between processes.

> **Read first:** JetStream concepts page (streams, consumers, retention
> policies, ack models) in the NATS docs. 20 minutes. The mental model:
> a *stream* persists messages on subjects; a *consumer* is a server-side
> cursor over the stream; `WorkQueue` retention deletes a message once acked —
> i.e., a durable job queue with redelivery for free.

---

## 1. The `checks` table

`sqlx migrate add create_checks` — design it yourself, target roughly:

```
id uuid pk, monitor_id uuid fk (on delete cascade),
status text check (status in ('up','down')),
http_status int null, latency_ms int null, error text null,
checked_at timestamptz not null default now()
```

Index `(monitor_id, checked_at desc)` — every dashboard query reads "latest N
checks for monitor".

## 2. The `Job` enum — the contract between processes

New crate `crates/jobs` (depends on core, db, bus). The *type* lives where
both enqueuer and worker can see it; put it in `core` or `jobs` — decide,
document.

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Job {
    RunCheck { monitor_id: Uuid },
    // NightlyCleanup, SendDigest — phases 7, 10
}
```

Serialized as JSON onto subject `jobs.dispatch`. The `tag = "type"` makes the
payloads debuggable with `nats` CLI — you'll value that within the hour.

## 3. Provision the stream (in `bus`)

**Your task:** a `bus::ensure_jetstream(client)` called from `bootstrap()`
that idempotently creates:

- stream `JOBS`: subjects `["jobs.>"]`, `retention: WorkQueue`,
  sensible `max_age` (e.g. 24h),
- durable **pull** consumer `workers` on it: `max_deliver: 5`,
  `ack_wait` ~30s, and a `backoff` schedule.

*Hint:* `async_nats::jetstream::new(client)`, then
`get_or_create_stream` / `get_or_create_consumer`. "get_or_create" is what
makes this safe to run from every process at boot — infrastructure as code,
no manual `nats stream add` in the runbook.

## 4. Enqueue from the server

**Your task:** `POST /commands/monitors/{id}/check_now` — publish the
serialized `Job` to `jobs.dispatch` **via JetStream publish** (so you get an
ack that it was persisted), return 204. Add the button to the monitor list
fragment.

Watch it sit there: `nats stream view JOBS`. No worker yet — the queue is
durable. Restart nats-server (`-sd` dir from phase 4 persists it) and it's
still there. *That's* why JetStream and not core pub/sub for jobs.

## 5. The worker binary

`cargo new apps/worker`. **Your task** — the whole binary is:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // tracing init, bootstrap() -> Ctx
    // get the `workers` pull consumer
    // let mut msgs = consumer.messages().await?;
    // while let Some(msg) = msgs.next().await {
    //     deserialize Job; jobs::handle(&ctx, job).await
    //     Ok  -> msg.ack()
    //     Err -> tracing::error! + msg.ack_with(AckKind::Nak(Some(delay)))
    // }
}
```

And `jobs::handle` for `RunCheck`:

1. load the monitor (gone? ack and skip — **don't** retry tombstones),
2. `reqwest` GET with a ~10s timeout (`cargo add reqwest --features rustls-tls`,
   default-features off; document why rustls in a comment),
3. measure latency, classify up/down, insert the `checks` row,
4. publish `events.checks.recorded` (subject constant!) so feeds update.

**Idempotency — read this twice.** JetStream is at-least-once: a crash after
the insert but before the ack ⇒ redelivery ⇒ the check runs twice. For
`RunCheck` a duplicate row is acceptable (it's a sample), and that's a
*decision to write down* in a comment. The discipline for every future job:
**either idempotent by construction, or deduplicated by key.** Phase 7 shows
the dedup-by-key technique.

## 6. Feed the dashboard

**Your task:** make the monitor list show latest status. Options: lateral
join / `distinct on (monitor_id)` for latest check per monitor — write it in
SQL, not in Rust. Subscribe the existing feed to `events.checks.>` as well
(*hint:* you can subscribe twice and `futures::stream::select`, or widen to
`events.>` and re-render — coarse is fine here, per phase 4's rule).

Add mise task `dev:worker` (watchexec like the server), add to `[tasks.dev]`
depends. `mise run dev` is now: nats + server + worker.

## 7. Commit

`jj describe -m "phase 6: jetstream work queue + worker, checks recorded"` · `jj new`

---

## Checkpoints

- [ ] "Check now" → status dot/latency appears in **both** browsers, server
      logs show no check activity, worker logs do.
- [ ] Stop the worker, click "Check now" 3×, `nats stream view JOBS` shows 3,
      start worker → drains, UI catches up.
- [ ] Point a monitor at a dead URL → `down` row with `error`, UI reflects it.
- [ ] Make `handle` return `Err` unconditionally (temporarily): observe
      nak/backoff redeliveries stopping at `max_deliver`. Revert.
- [ ] Run **two** workers (`cargo run -p worker` twice): jobs are split, not
      duplicated. (This is the durable consumer doing competing-consumers.)

## Stretch goals

- Graceful shutdown: `tokio::signal::ctrl_c` + finish in-flight job before
  exit. Template-worthy.
- A `failed_jobs` Postgres table written on terminal failure (max_deliver
  exhausted ⇒ JetStream advisory subjects — look up
  `$JS.EVENT.ADVISORY.CONSUMER.MAX_DELIVERIES`). Poor-man's DLQ.

Next: `07-scheduler.md` — the third process, and checks run themselves.
