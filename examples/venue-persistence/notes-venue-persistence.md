# QA notes — venue flow

Free-form observations, written the way a tester actually writes them: some are
real, some are guesses. The point of `checklist-enricher` is that it decides
which is which by reading the code, and reports the guesses instead of quietly
turning them into test scenarios.

---

- Saved venues seem to be capped at three. Worth checking what the fourth click
  actually does — I don't think the member is told anything.

- The venue you pick appears to be remembered between sessions now. Check what
  happens if that stored value is garbage.

- Venues are sorted by how busy they are. Two venues with the same number of
  meetings — what decides which comes first?

- I think anyone signed in can add a venue to the global list. If so, can they
  also rename an existing one?

- New venues added from the app might be missing their timezone. Meeting times
  could then display wrong for those venues.

- The profile screen and the browse screen both let you save a venue. Do they
  behave the same way?

- Does the app send a push notification when someone saves the same venue as
  you? Would be a nice growth feature.

- Meeting counts might include meetings that already finished.

- There's probably rate limiting on how many venues one account can add per day.
