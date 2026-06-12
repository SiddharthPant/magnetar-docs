# Phase 7 — Scheduler: time as an input

**Goal:** the third binary. Every monitor gets checked on its own
`interval_seconds` automatically; a nightly job prunes old checks. The
scheduler **only publishes jobs** — it never executes work. With dedup, two
scheduler replicas are safe.

**Concepts:** tick loops vs cron expressions, deterministic message IDs +
JetStream dedup window, separating "deciding when" from "doing".

---

## 1. Design before code

The scheduler answers one question per tick: *what is due?* Two patterns:

1. **Tick loop** (interval): every 15s, query Postgres for monitors whose
   last check is older than their interval, publish a `RunCheck` per due
   monitor. State lives in the DB; scheduler is stateless.
2. **Cron** (wall-clock): "03:00 daily prune", "Mon 09:00 digest". Use
   `tokio-cron-scheduler` (or hand-roll with `croner` + sleep — check both on
   crates.io, pick, document why).

Magnetar needs both, and the template should demonstrate both. That's the
real reason the domain was chosen.

## 2. Due-checks tick

`cargo new apps/scheduler`. **Your task:**

1. `bootstrap()`, then `tokio::time::interval(Duration::from_secs(15))` loop.
2. Each tick, one SQL query for due monitors. Write it yourself; the shape:

   ```sql
   select m.id from monitors m
   left join lateral (
     select checked_at from checks
     where monitor_id = m.id
     order by checked_at desc limit 1
   ) c on true
   where c.checked_at is null
      or c.checked_at < now() - make_interval(secs => m.interval_seconds)
   ```

   (Understand `left join lateral` — it's the "latest row per parent" tool
   you'll reuse constantly.)
3. Publish `Job::RunCheck` per due monitor — **with a deterministic
   `Nats-Msg-Id` header**. This is the phase's big idea:

   ```
   msg id = "run-check-{monitor_id}-{unix_minute_bucket}"
   ```

   JetStream drops duplicates with the same id inside the stream's
   `duplicate_window` (default 2 min — set it explicitly on the JOBS stream
   in `bus`). Consequences, in order of importance:
   - two scheduler replicas don't double-check anything → HA by accident;
   - a tick racing a slow previous check can't pile up duplicates;
   - "Check now" (phase 6) should **keep a random id** — manual = always runs.

   *Hint:* `jetstream.publish_with_headers(...)`, header name `Nats-Msg-Id`.

## 3. Nightly prune

**Your task:** add `Job::PruneChecks { keep_days: u32 }` handled by the
**worker** (one `delete ... where checked_at < now() - interval`), and a cron
entry in the scheduler (`0 0 3 * * *`) publishing it with msg id
`prune-checks-{yyyy-mm-dd}`. Note the pattern: scheduler decides *when*,
worker knows *how*. The scheduler binary has **no db-write code and no
business logic** — keep it that way forever.

## 4. Ops niceties

- mise: `dev:scheduler` task; add to `[tasks.dev]` depends. `mise run dev` =
  nats + server + worker + scheduler. Four processes, one command — the
  full template experience.
- Log every publish at `info` with monitor id + msg id. Quiet ticks at `debug`.
- Shorten a monitor's interval to 10–15s while developing so you can watch it
  breathe.

## 5. Commit

`jj describe -m "phase 7: scheduler — due-check ticks + nightly prune, msg-id dedup"` · `jj new`

---

## Checkpoints

- [ ] Create a monitor with `interval_seconds = 15`, touch nothing: checks
      appear forever, dashboard ticks live in two browsers.
- [ ] Run **two** schedulers simultaneously for 5 minutes: no duplicate
      checks (verify by counting rows per minute).
- [ ] Stop the worker for 2 minutes with the scheduler running: queue fills
      *without* unbounded duplicates per monitor (dedup window working),
      worker restart drains it.
- [ ] The prune job fires (test by setting the cron to `* * * * * *`
      temporarily) and deletes old rows.
- [ ] `rg -t rust 'sqlx::query' apps/scheduler` shows reads only.

## Stretch goals

- Jitter: spread due checks within the tick (±few seconds) so 200 monitors
  don't fire simultaneously. (Thundering herd, meet your template.)
- Emit a `scheduler.heartbeat` event each tick; surface "scheduler last seen"
  on an admin page in phase 9. Dead-scheduler detection for free.

Next: `08-auth.md` — time to find out who's clicking.
