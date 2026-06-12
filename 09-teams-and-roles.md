# Phase 9 — Teams, roles, and the owner panel

**Goal:** users belong to multiple teams; monitors belong to a team; team
**admins** manage members/invites/monitors, **members** view; **app owners**
get a global admin area. The sidebar grows a team switcher. This is the
multi-tenancy pattern you'll lift into every client project, so the *shape*
matters more than the features.

**Concepts:** membership join tables, authorization as typed extractors
("parse, don't validate" applied to permissions), tenancy scoping in every
query, invitation tokens.

---

## 1. Schema

`sqlx migrate add create_teams`. Target:

```
teams:        id, name, created_at
team_members: team_id fk, user_id fk, role text check (role in ('admin','member')),
              created_at, primary key (team_id, user_id)
invitations:  id, team_id fk, email citext, role, token_hash text unique,
              expires_at, accepted_at null, created_at
monitors:     + team_id uuid not null fk   -- the scoping migration
```

The monitors change needs care: existing rows need a team. Migration strategy
(do this deliberately, it's a client-project rehearsal): create a "Default
team", backfill, then add `not null`. Also: every user gets a personal team on
registration? Or team creation is explicit? **Decide, document in README.**
(Template recommendation: auto-create "{name}'s team" on register, owner =
admin — removes the empty-state cliff.)

## 2. Authorization as types

Extend the extractor pattern from phase 8. **Your task:**

```rust
pub struct TeamMember { pub user: User, pub team: Team, pub role: Role }
pub struct TeamAdmin  (pub TeamMember);   // constructed only if role == Admin
pub struct AppOwner   (pub User);         // only if is_app_owner
```

Each implements `FromRequestParts<Ctx>`: `TeamMember` reads the active team
(see §3) + membership row in one query; `TeamAdmin` wraps it and rejects
non-admins with 403; `AppOwner` checks the flag. Now signatures *are* the
authz spec:

```rust
async fn delete_monitor(admin: TeamAdmin, Path(id): Path<Uuid>, ...) // admins only
async fn list_monitors(member: TeamMember, ...)                      // any member
async fn owner_users(owner: AppOwner, ...)                           // owner panel
```

If a handler compiles with `TeamMember` it cannot accidentally allow admin
actions. This is the phase's biggest idea — authorization you can grep.

## 3. Active team + switcher

Where does "current team" live? Options: path prefix (`/t/{team_id}/...`),
session column, or cookie. **Template stance: path prefix.** It's
bookmarkable, it makes the tenancy explicit in every URL, and the extractor
reads `Path` — no hidden state. (Write the tradeoff down: a `current_team_id`
session column is fewer characters in URLs but invisible state.)

**Your task:** restructure protected routes under `/t/{team_id}`, build the
sidebar team switcher (list memberships, links to same page in other team),
and a `POST /commands/teams/create`.

## 4. Scope every query — the tenancy drill

**Your task:** every monitor/check query gains `where team_id = $1` (or a
join that pins it). Then the drill that makes it stick: log in as a member of
team A, take a monitor id from team B, hit
`/t/{teamA}/monitors/{teamB_monitor}` and every command with it. Everything
must 404 (not 403 — don't confirm existence). *Hint:* `... where id = $1 and
team_id = $2` returning `Option` → `None` ⇒ 404. **Feeds too:** the feed for
team A must subscribe/render only team A's data — and now your event subjects
earn granularity: `events.team.{team_id}.monitors.changed`. Update
`subjects.rs` with builder fns instead of bare consts.

## 5. Invitations

**Your task:**

- Admin form: email + role → `POST /commands/teams/{id}/invite`: random token
  (CSPRNG), store **hash**, 7-day expiry, enqueue `Job::SendInviteEmail` —
  worker "sends" by logging the accept-URL (a `Mailer` trait with a `LogMailer`
  impl; a real SMTP impl is a client-project concern, the *seam* is the
  template's job).
- `GET /invite/{token}` (public tree): valid+unexpired → if logged in, join &
  redirect; else send to register with the invite carried through (decide
  the mechanics — query param survives the form round-trip? cookie? document).
- Team settings page: members list with roles, pending invites, revoke
  (admin), remove member (admin), leave team (self, unless last admin —
  enforce!).

## 6. Owner panel

`/admin` (the `AppOwner` extractor): all teams w/ member+monitor counts, all
users, suspend toggle (`users.suspended_at`; the `CurrentUser` extractor
rejects suspended). Plus the phase-7 stretch if you did it: scheduler last
heartbeat. Keep it one page — it's a template, not a product.

## 7. Commit

`jj describe -m "phase 9: teams, roles, invitations, owner panel"` · `jj new`

---

## Checkpoints

- [ ] One user in two teams sees different monitor lists; switcher works; the
      cross-tenant drill (§4) yields 404s everywhere, **including commands
      fired via curl with a forged team in the path**.
- [ ] Member sees no delete/invite buttons *and* the commands reject them
      server-side (never trust hidden UI).
- [ ] Invite flow round-trips: invite → log shows URL → accept as new user →
      member appears live (NATS!) in the admin's open settings page.
- [ ] Last admin can't leave/demote; owner panel inaccessible to non-owners.
- [ ] Live updates are team-scoped: activity in team B doesn't patch team A's
      open dashboard (watch the network tab).

## Stretch goals

- A `#[sqlx::test]` integration test for the cross-tenant 404 drill — the
  single highest-value test in the whole template.
- Audit-ish log: `team_events(team_id, actor_id, kind, at)` written on
  membership changes. Cheap, clients always ask for it eventually.

Next: `10-incidents-and-polish.md` — make the dashboard worth screenshotting.
