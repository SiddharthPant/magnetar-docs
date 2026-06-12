# Phase 10 — Incidents, sparklines, toasts: the demo phase

**Goal:** the worker opens/resolves incidents automatically; the monitor
detail page gets a live latency sparkline and paginated check history; a
toast system gives command feedback. This is the phase that makes the
two-browser demo land with clients.

**Concepts:** derived state with concurrency control, fragments as read-model
projections, granular re-render scope (finally earning it), append-mode
patches, the form-validation pattern formalized.

---

## 1. Incidents

`sqlx migrate add create_incidents`:

```
incidents: id, monitor_id fk, started_at, resolved_at null, cause text
-- partial unique index: only ONE open incident per monitor:
create unique index one_open_incident on incidents (monitor_id)
  where resolved_at is null;
```

**Your task:** extend the worker's `RunCheck` handler:

- check is `down` and no open incident → open one,
- check is `up` and an open incident exists → resolve it,
- publish `events.team.{id}.incidents.changed` on transitions only.

The race to respect: two checks for the same monitor processed concurrently
(two workers!). The partial unique index makes "open" atomic —
`insert ... on conflict do nothing`. Reason through resolve as well
(`update ... where resolved_at is null` is naturally idempotent). This is
at-least-once delivery meeting derived state; the DB constraint is the
arbiter, not application logic. That sentence is template gospel.

UI: incidents page per team (open + recent resolved, duration), red banner on
the dashboard while any incident is open — live, of course.

## 2. Monitor detail page + sparkline

`GET /t/{team}/monitors/{id}` — **Your task:**

- header (name, url, current status, uptime % over 24h — one SQL aggregate,
  write it with `filter (where ...)`),
- a latency **sparkline of the last ~50 checks as inline SVG from Askama**:
  a `<polyline>`, points computed in a small pure Rust fn
  (`Vec<(x,y)> → "x1,y1 x2,y2 ..."`) — unit-test it with fixture data. No
  chart library. This is a flex *and* a load-bearing template decision:
  zero JS deps, server-rendered, patches like any fragment.
- paginated check table (`?before=<checked_at>` keyset pagination — not
  OFFSET; write down why: stable under inserts, index-friendly).

**Granular re-rendering, earned:** this page updates every few seconds per
monitor. Re-rendering the whole page fragment is wasteful and would fight
pagination state. Give the page its own feed
(`/t/{team}/feeds/monitors/{id}`) subscribed to
`events.team.{t}.monitor.{id}.checks` and patch **only** `#status-header` and
`#sparkline`. The worker now publishes both the granular and the coarse team
subject — or better, publish granular only and have the list feed subscribe
with a wildcard (`events.team.{t}.monitor.*.checks`). Choose; document.

## 3. Toasts

**Your task:** a `#toasts` region in the layout + a `web::toast(level, msg)
-> PatchElements` helper using **append mode** (PatchElements supports modes
other than morph — find append in the SDK) with a `data-on-load` self-removal
timer or a small `data-signals` TTL trick. Use it for: invite sent, monitor
created, command failures. Commands now optionally return a toast patch
instead of a bare 204 — note the rule that keeps CQRS honest: **toasts are
feedback about the request, never the data itself.** Data still flows through
feeds.

## 4. Form validation, formalized

You've patched error slots ad hoc since phase 3. **Your task:** standardize —
a `FormErrors` type rendering to `fragments/form_errors.html`, commands return
`422` + error patch on invalid input, fields keep values via signals. Apply
to: monitor create, invite, register/login. Write the pattern up in the root
README (it's the answer to "how do forms work in magnetar").

## 5. Sidebar polish

Real layout pass: sidebar (team switcher, nav: Dashboard / Incidents /
Settings / owner-only Admin), active-state highlighting, user block + logout.
Styling: pick **one** — hand-rolled CSS on the existing stylesheet, or
Tailwind via standalone CLI (`mise` can install `tailwindcss`; adds a build
step + watch task). Template stance suggestion: hand-rolled with CSS
variables — zero build step beats utility classes in a *template*. Your call;
README it.

## 6. Commit

`jj describe -m "phase 10: incidents, sparkline, toasts, validation pattern"` · `jj new`

---

## Checkpoints

- [ ] Kill a monitored URL: incident opens (banner in both browsers), restore
      it: auto-resolves with duration shown. Open incidents per monitor never
      exceed 1 — try to break it with two workers + 5s intervals.
- [ ] Detail page sparkline visibly ticks; paginating to page 3 doesn't get
      stomped by live patches.
- [ ] Toasts appear, stack, and self-dismiss; a failed command produces an
      error toast, not silence.
- [ ] All forms share the same validation UX; values survive a failed submit.
- [ ] Demo dry-run: two browsers, create monitor → live; kill URL → incident;
      invite member → joins live. Under 3 minutes, no narration needed.

## Stretch goals

- Uptime bars (90-day, GitHub-contribution style) per monitor — pure
  Askama/SVG again, one `generate_series` query.
- `data-on-intersect` lazy-loading for the check table.

Next: `11-production.md` — make it shippable, and the template is done.
