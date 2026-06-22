# Phase 3 — Postgres + SQLx: the monitors table

**Goal:** Postgres running in Docker, sqlx migrations, and the first real
domain object — `monitors` — with a list page and create/delete commands.
Still no auth, no teams; a monitor is just a row anyone can make.

**Concepts:** migrations as the schema's source of truth, compile-time checked
queries, connection pooling, the command → write → re-render loop (manually,
for now — NATS automates the re-render in phase 4).

**Status:** complete in `/Users/sid/projects/playground/magnetar`. The database
container is running, the `create_monitors` migration is installed, `\d monitors`
matches the migration shape, `cargo check` passes, the broken-query test confirms
SQLx catches schema mistakes at build time, and manual smoke tests verified the
`/monitors` page plus create/invalid-url/delete Datastar flows. Remaining
follow-up: commit the completed phase with the command in section 5, then move on
to phase 4.

---

## 1. Postgres in Docker

Create `compose.yml`:

```yaml
services:
  db:
    image: postgres:18.4-alpine3.23
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_db
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql]
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-app_db}",
        ]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  pgdata:
```

Add mise tasks:

```toml
[tasks."db:up"]
run = "docker compose up -d db"

[tasks."db:down"]
run = "docker compose down db"

[tasks."db:prompt"]
run = "docker compose exec db psql -U postgres -d app_db"

[tasks."db:migrate"]
run = "sqlx migrate run"
```

## 2. SQLx setup + first migration

```sh
cargo add sqlx --features postgres,runtime-tokio,uuid,time,migrate
cargo add uuid --features v7,serde
cargo add time --features serde
```

Make sure `.env` matches the compose credentials:

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app_db
```

```sh
sqlx database create        # uses DATABASE_URL from .env via mise
sqlx migrate add create_monitors
```

**Your task:** write the migration. Design it yourself first, then compare:

- `id uuid primary key` — Postgres 18 has `uuidv7()` built in, so you can use
  `DEFAULT uuidv7()` in the schema. If you ever downgrade to PG 17 or earlier,
  generate `Uuid::now_v7()` in Rust instead.
- `name text not null`
- `url text not null`
- `interval_seconds int not null default 60`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()` (stretch goal)

Example migration:

```sql
CREATE FUNCTION update_updated_at_column () RETURNS trigger AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ language plpgsql;

CREATE TABLE monitors (
    id uuid PRIMARY KEY DEFAULT uuidv7 (),
    name TEXT NOT NULL,
    url TEXT NOT NULL,
    interval_seconds int NOT NULL DEFAULT 60,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TRIGGER update_monitors_updated_at
    BEFORE UPDATE ON monitors
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column ();
```

Run `sqlx migrate run`, verify in `mise run db:prompt` with `\d monitors`.

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
    "select id, name, url, interval_seconds, created_at, updated_at
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

- [x] Fresh clone test: `rm`-nothing, but mentally — `mise run db:up`,
      `sqlx database setup`, `mise run dev:server` is the whole bring-up.
- [x] Creating a monitor updates the list without reload and clears the form.
- [x] Invalid URL shows an inline error, no row written.
- [x] A wrong column name in `query_as!` fails the **build**.
- [x] `\d monitors` matches your migration exactly.

## Stretch goals

- The example migration above already includes `updated_at` + a Postgres trigger
  to maintain it. If you skipped it, add it now — you'll thank yourself.
- Try `query_as::<_, Monitor>()` (runtime, FromRow) for one query to feel the
  difference, then change it back to the macro and keep the macro rule:
  **macros by default**, runtime API only for genuinely dynamic SQL.

Next: `04-nats-live-updates.md` — the moment magnetar becomes multiplayer.
