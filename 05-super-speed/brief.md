# Rook Dispatch — Product Brief
## Recovering after a quiet stretch

**Prepared for:** Helen Achebe, Director of Product | **By:** PM, Dispatch | **22 Sept 2026**
**Status:** Direction for alignment — not an implementation spec

---

### The problem

Since 4.2 (12 Aug), a responder who *misses* an offer by a few seconds is scored exactly the same as one who *refuses* it — and nothing ever tells the two apart again. **Vesper** (handled by Aunt Dot) is the clearest case: a steady 13–15 pings/week responder for six weeks straight, down to 1/week by month's end after one unlucky stretch. She never filed a ticket — three other responders (Meteor Mite, The Undertow, Farlight) show the same shape.

### The direction

**Same speed. Fairer memory. No permanent lockouts.**
- Keep the 60-second window — it's not the problem, and responders asked for the faster routing it enables.
- Stop scoring a missed offer the same as a refusal.
- Let quiet time itself start working back in a responder's favor — recovery shouldn't depend on lucky timing.

**Why not just revert the timeout:** it slows the problem, it doesn't fix it — the scoring gap predates 4.2 (Wen flagged it in a code comment years ago, never resolved) and reverting does nothing for responders already locked out. It also gives up a change responders had been asking for.

### Five decisions, resolved

| Open question | Direction |
|---|---|
| Does Kip *see* something change? | No new control or dashboard needed for v1 — he notices because pings resume, not because of a label. A visible indicator is a fast-follow design question, not a blocker. |
| How fast should recovery feel? | Target: back to normal standing within 2–3 weeks of going quiet — roughly the same speed it took to fall. |
| Does Meteor Mite get fixed today? | Yes — currently-stuck responders get a one-time correction alongside the new ongoing mechanism. Nobody waits through the new process on top of the month they've already lost. |
| Do we also fix over-served responders (The Gale)? | Yes — the same mechanism pulls both ends back toward normal. Not extra scope; same lever. |
| How do we know it's working, given the worst-hit never complain? | Add a standing check independent of tickets — an automatic flag for any responder under a low-ping threshold for several weeks, not reliant on someone filing a complaint. |

### Non-goals

Not reverting the timeout · not touching the 4.2 proximity/acceptance rebalance · not a manual per-responder support fix · not a new handler-facing setting or toggle.

### Success criteria

- No responder reaches near-total exclusion and stays there with no path back.
- Spread of weekly volume across responders returns to pre-4.2 baseline — tracked directly, not inferred from the (misleading) aggregate acceptance rate.
- Currently-stuck responders (Vesper, Meteor Mite, The Undertow, Farlight) are confirmed recovered, not just the average.

**Evidence:** `director-request.txt` · `dispatch-routing/history.py` (mechanism + Wen's 2019 note) · `callout-history.csv` (the divergence) · `feedback/tickets/` + `feedback/interviews/` (the ticket blind spot).
