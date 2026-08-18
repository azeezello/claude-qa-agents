# Test checklist — feature/venue-persistence

**Branch:** feature/venue-persistence | **Project:** readclub | **Stack:** React 19 + Vite SPA · Postgres with row-level security · serverless functions | **Date:** 2026-08-18

> Sample output. `readclub` is a fictional reading-groups app: members pick a
> venue, browse upcoming meetings there, and save a few favourite venues to
> their profile. The branch under test made venue selection persistent and let
> signed-in members add new venues to a catalogue everyone reads.

---

## Summary

Venue selection used to be throwaway state: a visitor picked a venue, and it was
forgotten on refresh. This branch makes location sticky and shared. The chosen
venue now survives a page reload, and a signed-in member's saved venues are
stored on their profile instead of only in the browser, so they follow the
account across devices. The venue list itself is no longer a fixed catalogue —
when someone picks a venue that isn't known yet, the app adds it for everyone,
and a new permission rule allows signed-in members to do that. Finally, the
list is reordered: venues with the most upcoming meetings now appear first, and
each one shows how many are coming up, replacing the previous fixed ordering.

---

## Priorities

| Priority | Flows | Rationale |
|---|---|---|
| 🔴 Critical | F2, F4 | F2 writes to the member's stored profile; F4 lets ordinary members write to a catalogue every visitor reads — a new write permission on shared data |
| 🟠 High | F1, F3 | F1 is the most visible behaviour change for every visitor; F3 changes what the whole venue list looks like and can silently show wrong counts |
| 🟡 Medium | F5 | Meeting creation and editing pick up venue changes indirectly |
| 🟢 Low | F6 | The shareable meeting card was refactored — no intended behaviour change, needs a regression pass only |

---

## Regression scope

| Flow at risk | Why | Automated? |
|---|---|---|
| Browse feed — venue filter and radius | The venue list loader changed shape and ordering; every screen that reads it inherits the change | [AUTO] partial — service unit tests only, no UI test |
| Map screen — pins and venue switching | Now receives shared venue handlers instead of its own local ones | [MANUAL] |
| Profile screen — saved venues list | Saving now also writes to the shared venue catalogue | [MANUAL] |
| Meeting creation | Adds a catalogue write before creating the meeting; a failure there must not block creation | [MANUAL] |
| Meeting editing | Venue resolution for the edit form now also searches saved venues | [MANUAL] |
| Shareable meeting card | Switched to a different venue lookup source during cleanup | [MANUAL] |
| First paint after sign-in | The delay before loading saved venues was removed — profile load now races the initial render | [MANUAL] |

---

## Test data

| Entity | Required state | Example |
|---|---|---|
| Visitor | signed out, empty browser storage | incognito window |
| Member A | signed in, no saved venues | fresh account |
| Member B | signed in, exactly 3 saved venues | at the cap |
| Venue — known | already in the catalogue | any seeded venue |
| Venue — unknown | not in the catalogue, reachable by search | any small library or café |
| Venue — busy | 3+ upcoming meetings | the main branch library |
| Venue — empty | 0 upcoming meetings | any unused venue |
| Meeting — past | starts in the past, so it must not be counted | yesterday |
| Meeting — cancelled | not active, so it must not be counted | any non-active status |

---

## Flow 1 — Visitor picks a venue and it sticks

*Trigger: venue picker on Browse and Map*
*Expected: the selected venue survives a page refresh and a browser restart, and never leaves the app blank*

| ID | Pri | Precondition | Scenario | What to check | Result | Notes |
|---|---|---|---|---|---|---|
| F1-01 | 🟠 | Signed out, empty storage | Open the app for the first time | A default venue is selected and the feed loads |  | [MANUAL] |
| F1-02 | 🟠 | Signed out | Pick a different venue, then refresh the page | The same venue is still selected after reload |  | [MANUAL] |
| F1-03 | 🟠 | Signed out | Pick a venue, close the browser entirely, reopen the app | The venue is still selected |  | [MANUAL] |
| F1-04 | 🟠 | Signed out | Pick a venue on Browse, switch to the Map screen | The Map shows the same venue |  | [MANUAL] [REGRESSION] |
| F1-05 | 🟠 | Signed out | Pick a venue on the Map, switch to Browse | Browse shows the same venue |  | [MANUAL] [REGRESSION] |
| F1-06 | 🟡 | Stored venue value manually corrupted to invalid text | Reload the app | The app still opens on a valid default venue instead of an error screen |  | [MANUAL] |
| F1-07 | 🟡 | Stored venue refers to a venue removed from the catalogue | Reload the app | The app opens without a blank feed; the stale venue is handled gracefully |  | [MANUAL] |
| F1-08 | 🟡 | Browser storage disabled or full (private mode, quota exceeded) | Pick a venue and refresh | The app keeps working for the session; no error is shown to the user |  | [MANUAL] |
| F1-09 | 🟢 | Two tabs open on the app | Change the venue in tab 1, then refresh tab 2 | Tab 2 picks up the venue chosen in tab 1 |  | [MANUAL] |

---

## Flow 2 — Signed-in member saves and removes venues

*Trigger: Save / ✕ in the venue picker, on Browse, Map and Profile*
*Expected: saved venues are stored on the account, survive sign-out, and are capped at three*

| ID | Pri | Precondition | Scenario | What to check | Result | Notes |
|---|---|---|---|---|---|---|
| F2-01 | 🔴 | Member A signed in, no saved venues | Save a venue from the Browse picker | The venue shows a SAVED marker immediately |  | [MANUAL] |
| F2-02 | 🔴 | Follows F2-01 | Sign out, sign back in as Member A | The saved venue is still there |  | [MANUAL] |
| F2-03 | 🔴 | Follows F2-01 | Open the app on a different device as Member A | The saved venue is present there too |  | [MANUAL] |
| F2-04 | 🟠 | Member A has 2 saved venues | Save a third venue | All three appear as saved |  | [MANUAL] |
| F2-05 | 🔴 | Member B has exactly 3 saved venues | Save a fourth venue | Verify what the member is told. Either the fourth is accepted and one is dropped, or it is refused with a visible message — silently discarding the click is a defect |  | [MANUAL] boundary: cap = 3 |
| F2-06 | 🟠 | Follows F2-05 | Sign out and back in | The saved list matches exactly what was shown on screen before signing out |  | [MANUAL] boundary: cap = 3 |
| F2-07 | 🟠 | Member A has 1 saved venue | Remove it | It disappears from the saved list and stays gone after a refresh |  | [MANUAL] |
| F2-08 | 🟠 | Member A has saved venues | Save the same venue twice in a row | It appears once, not twice |  | [MANUAL] |
| F2-09 | 🔴 | Member A signed in, network throttled or offline | Save a venue | The member is not left believing a venue was saved when it was not — either it persists or a failure is visible |  | [MANUAL] |
| F2-10 | 🟠 | Member A saved a venue from Profile | Open Browse | The same saved venue is listed there |  | [MANUAL] [REGRESSION] |
| F2-11 | 🟠 | Member A saved a venue from the Map | Open Profile | The same saved venue is listed there |  | [MANUAL] [REGRESSION] |
| F2-12 | 🟡 | Signed out visitor | Try to save a venue | Behaviour is consistent and explained — either saving is unavailable or it prompts to sign in; it must not appear to succeed and then vanish |  | [MANUAL] |
| F2-13 | 🟡 | Member A signed in | Sign in and immediately interact with the venue picker before the page settles | Saved venues load without the list flickering back to empty or dropping a venue just saved |  | [MANUAL] [REGRESSION] |

---

## Flow 3 — Venue list ordering and meeting counts

*Trigger: opening the venue picker*
*Expected: venues are ordered by how many upcoming meetings they have, and the count shown matches reality*

| ID | Pri | Precondition | Scenario | What to check | Result | Notes |
|---|---|---|---|---|---|---|
| F3-01 | 🟠 | Venues with varying numbers of upcoming meetings | Open the venue picker | The busiest venue is at the top; the list descends by count |  | [MANUAL] |
| F3-02 | 🟠 | Two venues with the same number of meetings | Open the venue picker | They are ordered alphabetically relative to each other |  | [MANUAL] |
| F3-03 | 🟠 | A venue with 0 upcoming meetings | Open the venue picker | It appears with a count of 0 rather than being hidden |  | [MANUAL] boundary: count = 0 |
| F3-04 | 🔴 | A venue whose only meeting is in the past | Open the venue picker | The past meeting is not counted |  | [MANUAL] boundary: time = now |
| F3-05 | 🔴 | A meeting starting within the next few minutes | Open the venue picker | It is counted as upcoming |  | [MANUAL] boundary: time = now |
| F3-06 | 🟠 | A venue with a cancelled or non-active meeting | Open the venue picker | The inactive meeting is not counted |  | [MANUAL] |
| F3-07 | 🟠 | Signed out visitor | Open the venue picker | Counts are visible without signing in |  | [MANUAL] |
| F3-08 | 🟡 | Catalogue temporarily unreachable | Open the venue picker with the backend blocked | A usable built-in venue list is shown instead of an empty picker |  | [AUTO] service-level fallback is unit-tested; the on-screen result is not |
| F3-09 | 🟡 | Catalogue returns no venues at all | Open the venue picker | The built-in list is shown |  | [AUTO] |
| F3-10 | 🟡 | Catalogue reachable and populated | Open the venue picker | Real catalogue venues are shown, not the built-in list |  | [AUTO] |
| F3-11 | 🟠 | Previously the list had a fixed manual order | Compare the picker order against the old release | The change in order is intentional and no venue has disappeared from the list |  | [MANUAL] [REGRESSION] |
| F3-12 | 🟡 | A saved venue that also appears in the main list | Open the venue picker | The count shown in the saved section matches the count in the main list |  | [MANUAL] |

---

## Flow 4 — A new venue enters the shared catalogue

*Trigger: selecting or saving a venue that is not yet in the catalogue; creating a meeting*
*Expected: signed-in members can add a venue; signed-out visitors cannot; nobody can alter an existing one*

| ID | Pri | Precondition | Scenario | What to check | Result | Notes |
|---|---|---|---|---|---|---|
| F4-01 | 🔴 | Member A signed in; target venue absent from the catalogue | Save that venue | It is added to the catalogue and visible to other members afterwards |  | [MANUAL] |
| F4-02 | 🔴 | Signed-out visitor; target venue absent | Attempt the same action | The catalogue is not modified |  | [MANUAL] security |
| F4-03 | 🔴 | Member A signed in; venue already in the catalogue | Save it again | The existing entry is unchanged — no duplicate, no overwritten name or coordinates |  | [MANUAL] security |
| F4-04 | 🔴 | Member A signed in | Attempt to change an existing venue's name or coordinates through the app's own requests | The change is rejected; the catalogue entry stays as it was |  | [MANUAL] security |
| F4-05 | 🔴 | Member A signed in | Add a venue whose name contains script tags, emoji, and right-to-left text | The name is stored and displayed as literal text everywhere it appears, with no markup interpreted |  | [MANUAL] security |
| F4-06 | 🟠 | Member A signed in | Add a venue with an extremely long name | Either it is rejected with a clear message, or it displays without breaking the picker layout |  | [MANUAL] boundary |
| F4-07 | 🟠 | Member A signed in | Add a venue with coordinates at the extremes of the valid range | The venue is placed correctly on the map |  | [MANUAL] boundary |
| F4-08 | 🟠 | Member A signed in | Add a venue with missing or non-numeric coordinates | It is either rejected or handled without the map failing to render |  | [MANUAL] boundary |
| F4-09 | 🔴 | Member A signed in, catalogue write blocked or failing | Save a venue | The member's own saved-venue list still updates and no error surface breaks the screen |  | [MANUAL] |
| F4-10 | 🟠 | Two members signed in on separate devices | Both add the same new venue at the same moment | Exactly one catalogue entry results; neither member sees an error |  | [MANUAL] edge |
| F4-11 | 🟡 | Member A adds a new venue | A different member opens the picker | The new venue is present for them, ordered by its meeting count like any other |  | [MANUAL] |
| F4-12 | 🟠 | Fresh environment, migrations applied twice | Apply the release migrations to a database that already has them | Applying them again does not fail |  | [MANUAL] deployment |

---

## Flow 5 — Creating and editing a meeting at a venue

*Trigger: Create screen submit; Edit screen open*
*Expected: the venue is added to the catalogue as a side effect, and a failure there never blocks the meeting*

| ID | Pri | Precondition | Scenario | What to check | Result | Notes |
|---|---|---|---|---|---|---|
| F5-01 | 🟠 | Member A signed in, venue not in the catalogue | Create a meeting at that venue | The meeting is created and the venue appears in the catalogue |  | [MANUAL] |
| F5-02 | 🔴 | Member A signed in, catalogue write failing | Create a meeting | The meeting is still created successfully |  | [MANUAL] |
| F5-03 | 🟠 | Member A has a meeting at a venue in the built-in list | Open it for editing | The venue field is pre-filled correctly |  | [MANUAL] [REGRESSION] |
| F5-04 | 🟠 | Member A has a meeting at a venue only present in their saved list | Open it for editing | The venue field is pre-filled with that venue, not silently replaced by the currently selected one |  | [MANUAL] |
| F5-05 | 🟡 | Member A has a meeting at a venue in neither list | Open it for editing | The form opens with a sensible venue rather than crashing or showing an empty field |  | [MANUAL] |
| F5-06 | 🟡 | Follows F5-01 | Check the new venue's count in the picker | It reflects the meeting just created |  | [MANUAL] |

---

## Flow 6 — Shareable meeting card regression

*Trigger: sharing a meeting as an image*
*Expected: unchanged output — this flow was only refactored*

| ID | Pri | Precondition | Scenario | What to check | Result | Notes |
|---|---|---|---|---|---|---|
| F6-01 | 🟢 | A meeting at a built-in venue | Generate the share card | The venue name renders correctly on the image |  | [MANUAL] [REGRESSION] |
| F6-02 | 🟠 | A meeting at a newly added venue | Generate the share card | The venue name renders rather than appearing blank or as a raw identifier |  | [MANUAL] [REGRESSION] |
| F6-03 | 🟢 | Any meeting | Compare the generated card against the previous release | Layout, rounded corners and spacing are unchanged |  | [MANUAL] [REGRESSION] |

---

## Exploratory sessions

| Theme | Question to explore |
|---|---|
| Shared catalogue as an attack surface | Every signed-in member can now write to a table every visitor reads. What is the worst a single account can do to the venue list in five minutes, and how would anyone notice? |
| Time zones and the "upcoming" boundary | Counts depend on comparing meeting times to "now". What does a member in a time zone 12 hours ahead see, and does a meeting starting tonight count for both of them? |
| Account state transitions | What happens to the locally-stored venue and the profile-stored list when a member signs out, signs in as someone else, or deletes their account? |
| Ordering stability | The list reorders itself as meetings are created. Can a venue move under the member's finger between rendering the picker and tapping a row? |
| Offline and flaky networks | Saving a venue writes to two places. Walk through every combination of one succeeding and the other failing, and decide which ones the member should be told about. |
