# Verification guide — feature/venue-persistence

**Branch:** feature/venue-persistence | **Project:** readclub | **Date:** 2026-08-18
**Data:** mcp | **Logs:** mcp

> Sample output, paired with `checklist-venue-persistence.md`. The checklist
> says what to do; this guide says where to look afterwards. `flow-verifier`
> reads it to decide PASS / FAIL / INCONCLUSIVE.

---

## Flow scope

| Flow | Log sources | Tables | Endpoints |
|---|---|---|---|
| F1 | — (browser-only state) | — | — |
| F2 | postgres | `members` | `PATCH /rest/v1/members` |
| F3 | postgres | `venues`, `meetings` | `GET /rest/v1/venues`, `GET /rest/v1/meetings` |
| F4 | postgres | `venues` | `POST /rest/v1/venues` |
| F5 | postgres | `venues`, `meetings` | `POST /rest/v1/meetings` |
| F6 | — (browser-only rendering) | — | — |

F1 and F6 have no server-side evidence: selected-venue persistence lives in
browser storage and share-card rendering happens on a canvas. Verify them by
observation, not by query — `flow-verifier` will correctly return INCONCLUSIVE
for them, which is the honest answer, not a failure.

---

## Database

### venues

- **Schema:** public
- **Identifier column:** `id` (text slug, e.g. `central-library`)
- **Query:**
  ```sql
  SELECT id, name, city, lat, lng, timezone, sort_order
  FROM public.venues
  WHERE id = '<venue_id>';
  ```
- **What to check:**
  - after F4-01 a row exists for the newly added venue, with the name and
    coordinates that were shown on screen
  - after F4-03 and F4-04 the row is **byte-identical** to what it was before
    the attempt — the client insert ignores duplicates and no UPDATE policy
    exists, so an existing venue must never change
  - `timezone` may be null on a client-added venue; it is filled in later by the
    timezone backfill job
  - `sort_order` is no longer read by the app — a null or zero here is not a defect
- **Access rule (changed by this branch):**
  - `venues_public_read` — `SELECT` allowed for everyone, unchanged
  - `authenticated_can_insert_venues` — **new**: `INSERT` allowed when the
    request carries an authenticated user
  - there is **no** `UPDATE` and **no** `DELETE` policy, so both remain denied
    for every client role
- **Catalogue snapshot**, for comparing before and after a test:
  ```sql
  SELECT count(*) AS total,
         count(*) FILTER (WHERE timezone IS NULL) AS missing_tz
  FROM public.venues;
  ```

### members

- **Schema:** public
- **Identifier column:** `id` (uuid, equals the auth user id)
- **Query:**
  ```sql
  SELECT id, display_name, saved_venues
  FROM public.members
  WHERE id = '<member_id>';
  ```
- **What to check:**
  - after F2-01 `saved_venues` contains the saved venue
  - after F2-05 the array holds **at most 3 entries** — this is the cap the UI
    enforces; more than 3 here means the client cap was bypassed
  - after F2-07 the removed venue is gone from the array
  - the array contents match what the screen displayed — a mismatch is the
    exact bug F2-09 is looking for
  - each entry carries `id`, `name`, `city`, `lat`, `lng`; a bare string instead
    of an object means an older client wrote it

### meetings

- **Schema:** public
- **Identifier column:** `venue` (text, matches `venues.id`)
- **Query — the count the picker should be showing:**
  ```sql
  SELECT venue, count(*) AS upcoming
  FROM public.meetings
  WHERE status = 'active'
    AND starts_at >= now()
  GROUP BY venue
  ORDER BY upcoming DESC, venue;
  ```
- **What to check:**
  - the number next to each venue in the picker equals `upcoming` for that venue
  - the picker's order matches this query's order, with ties broken by the
    displayed venue **name** rather than by `venue` id — a venue whose slug and
    display name sort differently is the case that separates the two
  - a meeting with `starts_at` in the past does not appear (F3-04)
  - a meeting with a status other than `active` does not appear (F3-06)

---

## Log events

This branch adds no server-side logging: the venue insert and the profile update
are plain REST calls made from the browser, and both client call sites swallow
their errors so the member's own action is never blocked. There is therefore no
application log line to match on.

Verify these flows from database logs instead of application logs.

### Flow F4 — catalogue insert

Search the `postgres` log source for statements against `venues` during the test
window.

- A successful add produces an `INSERT INTO "public"."venues"` statement.
- A blocked add (F4-02, signed-out visitor) produces a row-level-security
  violation — `new row violates row-level security policy for table "venues"`.
  **Seeing this error for F4-02 is a PASS, not a FAIL** — it is the evidence the
  policy works. Judge it by which scenario is being verified.
- An attempt to modify an existing venue (F4-04) produces either no statement at
  all or a policy violation; what it must never produce is a successful `UPDATE`.

### Flow F2 — profile write

- A successful save produces an `UPDATE "public"."members" SET "saved_venues"`
  statement for the member's own id.
- An update touching a **different** member's id would be a serious finding; the
  profile policies should make it impossible.

### Flow F3 — read path

- Each picker open produces one `SELECT` against `venues` and one against
  `meetings`. Two selects rather than one is expected — they are issued in
  parallel.
- If the picker showed the built-in fallback list (F3-08), the `venues` select
  either failed or returned zero rows. Confirm which, because those are
  different defects.

---

## Endpoints

The app has no bespoke HTTP layer for these flows — the browser talks to the
auto-generated REST API directly. The relevant calls:

### GET /rest/v1/venues

- **When called:** every time the venue list loads
- **Auth:** public — works signed out
- **Expected:** HTTP 200 with the full catalogue
- **Note:** issued in parallel with the meeting count query; a failure of either
  one falls back to the built-in venue list rather than erroring

### GET /rest/v1/meetings?select=venue&status=eq.active&starts_at=gte.{now}

- **When called:** alongside the venue list load
- **Auth:** public
- **Expected:** HTTP 200 with one row per upcoming meeting

### POST /rest/v1/venues

- **When called:** saving a venue, or creating a meeting at one
- **Auth:** authenticated — this is the permission the branch adds
- **Expected:** HTTP 201 for a new venue; a duplicate is a no-op because the
  client asks for duplicates to be ignored
- **Errors:** HTTP 401/403 when signed out — expected behaviour for F4-02

### PATCH /rest/v1/members?id=eq.{member_id}

- **When called:** saving or removing a venue while signed in
- **Auth:** authenticated, own row only
- **Expected:** HTTP 204
- **Note:** the client ignores the result, so a failure here is invisible in the
  UI. That gap is exactly what F2-09 tests, and the only way to confirm it is to
  compare the on-screen list against the `members` query above.

---

## Async triggers

### venue timezone backfill

- **Fires when:** a row is inserted into `venues` without a `timezone` value
- **Observable effect:** `timezone` is populated shortly after the insert
- **What to check:** after F4-01, re-run the `venues` query a minute later and
  confirm `timezone` is no longer null. A permanently null value on
  client-added venues means the new insert path bypasses the backfill that
  seeded rows relied on.
