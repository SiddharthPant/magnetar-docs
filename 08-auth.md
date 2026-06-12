# Phase 8 — Auth: users, passwords, sessions

**Goal:** register, login, logout. DB-backed sessions in an httpOnly cookie.
Every page except `/login` and `/register` requires auth, enforced by
middleware, with the current user available to every handler via an extractor.

**Concepts:** argon2id hashing, session-table auth (vs JWT — and why server
state is the right call here), axum middleware + extractors as the authz
toolkit, secure cookie attributes.

> Design stance for the template: **server-side sessions, not JWTs.** You have
> Postgres in every request path already; revocation is a `delete`; no token
> refresh choreography. Write this rationale in the README — clients ask.

---

## 1. Schema

`sqlx migrate add create_users_sessions`. Design first, compare:

```
users:    id uuid pk, email citext unique not null,
          password_hash text not null, is_app_owner bool not null default false,
          created_at timestamptz
sessions: id uuid pk,            -- this IS the cookie token; uuid v4, not v7 (why?)
          user_id uuid fk on delete cascade,
          expires_at timestamptz not null, created_at timestamptz
```

(`citext` needs `create extension if not exists citext;` in the migration.
Answer the v4-not-v7 question for yourself: v7 leaks creation time and is
sequential — session tokens must be unguessable. Better still: a random
128-bit+ token generated via a CSPRNG, stored hashed — decide how far the
template goes and document it.)

## 2. Password hashing

`cargo add argon2 password-hash` (in `core` or a new `crates/auth` — your
call). **Your task:** two functions with tests:

```rust
pub fn hash_password(plain: &str) -> Result<String>;
pub fn verify_password(plain: &str, hash: &str) -> Result<bool>;
```

*Hint:* `Argon2::default()` (argon2id), `SaltString::generate(&mut OsRng)`,
`PasswordHash::new` + `verify_password`. Hashing is ~100ms **by design** —
call it via `tokio::task::spawn_blocking` in handlers so you don't stall the
runtime. Write a test proving wrong passwords fail.

## 3. Pages and commands

**Your task:**

- `GET /register`, `GET /login` — Askama pages, plain Datastar forms.
- `POST /commands/auth/register` — validate (email shape, password length ≥
  12), insert user, create session, set cookie, then **redirect**. SSE
  patches can't navigate; the Datastar way is an `ExecuteScript`/location
  patch or simply respond with a redirect — check the SDK for the idiomatic
  redirect helper and use it. First user registered becomes `is_app_owner`
  (bootstrap problem solved; document it).
- `POST /commands/auth/login` — verify, create session, cookie, redirect.
  Same error message for "no such user" and "bad password" (enumeration).
- `POST /commands/auth/logout` — delete session row, clear cookie, redirect.

Cookie: `cargo add tower-cookies` (or axum-extra's `CookieJar` — pick one).
Attributes non-negotiable: `HttpOnly`, `SameSite=Lax`, `Path=/`, `Secure` in
prod (make it env-driven), sensible `Max-Age` matching `expires_at`.

## 4. The `CurrentUser` extractor + middleware

The pattern the whole authz story builds on:

**Your task:**

1. A `CurrentUser(pub User)` struct implementing `FromRequestParts<Ctx>`:
   read cookie → load session join user → check `expires_at` → reject with
   redirect-to-login if anything fails.
   *Hint:* one query, `query_as!` with a join; do **not** hit the DB twice.
2. Protect routes by construction, not by remembering: split the router into
   a public `Router` (login/register/assets/healthz) and a protected `Router`
   where handlers take `CurrentUser`. Merge them. A protected handler without
   the extractor should *feel* wrong in review — consider a
   `route_layer(middleware::from_extractor::<CurrentUser>())` on the protected
   tree so even handlers that don't read the user are gated.
3. Feeds too! `/feeds/*` must require auth — SSE endpoints are just GETs and
   are forgotten **constantly**. Put them under the protected tree.

## 5. Session hygiene

**Your task:** add `Job::PruneSessions` (worker deletes expired), scheduled
nightly with a dated msg id — you have the machinery; this is one of those
"the template demonstrates the pattern twice so it sticks" moments.

Show the logged-in user's email + logout button in the sidebar layout.

## 6. Commit

`jj describe -m "phase 8: auth — argon2, db sessions, CurrentUser extractor"` · `jj new`

---

## Checkpoints

- [ ] Logged out, every app page and `/feeds/*` redirects to `/login`;
      `curl -i localhost:3000/feeds/monitors` proves it.
- [ ] Cookie in devtools: HttpOnly ✓ SameSite=Lax ✓.
- [ ] Logout, then back-button: no authed content renders.
- [ ] Delete your session row in psql mid-session → next action bounces you.
      (This is the revocation story JWTs make hard — note it in the README.)
- [ ] `cargo test` covers hash/verify round-trip.

## Stretch goals

- Rate-limit login attempts (`tower` has primitives; per-IP via
  `ConnectInfo`). Even a crude in-memory limiter documents intent.
- Email-token invite/magic-link plumbing — **don't build it yet**; phase 9's
  invitations will reuse whatever token design you sketch here. Sketch only.

Next: `09-teams-and-roles.md` — the multi-tenant heart of the template.
