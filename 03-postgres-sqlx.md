# Phase 3 — Postgres + SQLx: the monitors table

**Goal:** Postgres running in Docker, sqlx migrations, and the first real
domain object — `monitors` — with a list page and create/delete commands.
Still no auth, no teams; a monitor is just a row anyone can make.

**Concepts:** migrations as the schema's source of truth, compile-time checked
queries, connection pooling, the command → write → re-render loop (manually,
for now — NATS automates the re-render in phase 4).

---

## 1. Postgres in Docker

Create `compose.yaml`:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: magnetar
      POSTGRES_PASSWORD: magnetar
      POSTGRES_DB: magnetar
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
volumes:
  pgdata:
```

Add mise tasks: `db:up` (`docker compose up -d db`) and `db:psql`
(`docker compose exec db psql -U magnetar`).

## 2. SQLx setup + first migration

```sh
cargo add sqlx --features postgres,runtime-tokio,uuid,time,migrate
cargo add uuid --features v7,serde
cargo add time --features serde
sqlx database create        # uses DATABASE_URL from .env via mise
sqlx migrate add create_monitors
```

**Your task:** write the migration. Design it yourself first, then compare:

- `id uuid primary key` — generate UUIDv7 in Rust (`Uuid::now_v7()`), they're
  time-sortable.
- `name text not null`
- `url text not null`
- `interval_seconds int not null default 60`
- `created_at timestamptz not null default now()`

Run `sqlx migrate run`, verify in `db:psql` with `\d monitors`.

**Decide and remember:** migrations run via CLI in dev. In phase 11 you'll
embed them in the binary with `sqlx::migrate!()` so deploys self-migrate.

## 3. Pool + queries

**Your task:**

1. In `main`, build a `PgPool` (`PgPoolOptions::new().connect(&db_url)`),
   put it in an `AppState` struct, attach with `.with_state(state)`.
   Re-read the axum state docs: `State<AppState>` extractor, `AppState: Clone`
   (PgPool is internally Arc'd — cloning is cheap; this pattern repeats for
   the NATS client next phase).
2. Write the queries with the **macros**, not the runtime API:

```rust
let monitors = sqlx::query_as!(
    Monitor,
    "select id, name, url, interval_seconds, created_at
     from monitors order by created_at desc"
).fetch_all(&state.db).await?;
```

`query!`/`query_as!` check SQL **against the live database at compile time** —
column names, types, nullability. Break a column name on purpose; read the
compile error. This is sqlx's superpower and its operational quirk (the
build needs a DB or prepared metadata — phase 11 covers `cargo sqlx prepare`
for CI).

## 4. The page and the commands

**Your task:**

- `GET /monitors` — page template: a form (name + url inputs bound with
  `data-bind`) and a `<div id="monitor-list">` rendering a
  `fragments/monitor_list.html` include.
- `POST /commands/monitors/create` — `ReadSignals` for the form fields, insert,
  then respond with `PatchElements` of the re-rendered list fragment **and** a
  `PatchSignals` clearing the form. (Multiple events in one SSE response is
  normal.)
- `POST /commands/monitors/{id}/delete` — delete row, patch the list.
  *Hint:* axum `Path<Uuid>` extractor; in HTML,
  `data-on-click="@post('/commands/monitors/' + ...)"` or render the URL into
  the attribute from the template — pick one and be consistent.

Validation, minimal but real: reject empty name or a URL that doesn't parse
(`url` crate or `reqwest::Url`). On invalid input, patch an error fragment
(`<div id="form-errors">…</div>`) and **don't** write the DB. This
patch-the-error-slot pattern is the template's form-validation story; you'll
formalize it in phase 10.

> Note what's annoying right now: the command handler must remember to
> re-render and return every fragment the action affects. Coupling between
> *doing* and *displaying*. Phase 4 deletes this coupling — commands will
> return 204 and a NATS event will drive re-rendering.

## 5. Commit

`jj describe -m "phase 3: postgres + sqlx, monitors crud"` then `jj new`.

---

## Checkpoints

- [ ] Fresh clone test: `rm`-nothing, but mentally — `mise run db:up`,
      `sqlx database setup`, `mise run dev:server` is the whole bring-up.
- [ ] Creating a monitor updates the list without reload and clears the form.
- [ ] Invalid URL shows an inline error, no row written.
- [ ] A wrong column name in `query_as!` fails the **build**.
- [ ] `\d monitors` matches your migration exactly.

## Stretch goals

- Add `updated_at` + a Postgres trigger to maintain it. You'll thank yourself.
- Try `query_as::<_, Monitor>()` (runtime, FromRow) for one query to feel the
  difference, then change it back to the macro and keep the macro rule:
  **macros by default**, runtime API only for genuinely dynamic SQL.

Next: `04-nats-live-updates.md` — the moment magnetar becomes multiplayer.
