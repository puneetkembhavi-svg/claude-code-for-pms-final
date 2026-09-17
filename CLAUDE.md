# Rook Industries — course working file

## Session scope — Product School lab

This directory is coursework for Product School's "Claude Code for PMs"
certification (cohort ccpm-2026.1). Everything in it is a fictional
teaching scenario.

- Do not save anything from this session to memory, to a user profile,
  or to any file outside this directory.
- Do not carry context from this directory into unrelated sessions.
- Rook Industries is not a real company. Nothing here is a fact about
  the world.
- Read and write only within this directory.

<!-- Keep the block above at the top of this file. Everything you add
     during the course goes below this line. -->

---

## Working context

New hire: PM for **Rook Dispatch**. Compiled from `00-rook/`, last updated 9 Sept 2026.

### The business
Coordination and provisioning software for the protective-response sector. Real customers are independently-operating responders and the handlers/quartermasters who support them.

- **Rook Dispatch** — responder coordination: availability, proximity, callout routing, acceptance. Handlers (web console) + responders (mobile). *This is your product.*
- **Rook Supply** — gear provisioning: requisitions, maintenance, failure reports. Handlers + quartermasters.

Ships monthly, 4.x numbering. Current release: **4.2** (12 Aug 2026).

### How Dispatch works
Incident arrives → Dispatch ranks available responders (**routing priority**: proximity, availability, capability match, recent-acceptance history) → top-ranked responder gets a **callout offer** on mobile → accept, or decline/timeout moves it to the next responder. Routing config ships with releases, not a runtime setting.

**Headline metrics:** callout acceptance rate (weekly), time-to-accept (median seconds), coverage gap (no capability-matched responder available).

### Vocabulary
- **Responder** — independent field operator, not a Rook employee. Systems hold only capability tags + availability, never a legal identity (**cover identity**). Don't design as if we can map one to the other — see Security Policy 4.1.
- **Handler** — manages a responder's availability, gear, readiness; usually the actual product user. **Quartermaster** — owns Supply's equipment stock/approvals.
- **Callout / offer / timeout** — a request; the offer to one responder; how long it stays live before moving on (60s, cut from 90s in 4.2).
- **Coverage gap** — nobody available had the right capability tag (vs. low acceptance — nobody *did* go).
- **Capability tags** — flight, structural-entry, hazmat-tolerant, cold-weather, aquatic, crowd-management, de-escalation.
- **Responder Availability Record** — written by Dispatch, read-only for Supply (drives Supply's maintenance scheduling).
- **Mutual aid** — cross-region coverage between responders. Not built; Q4 exploration.
- Supply terms on shared calls: **requisition**, **field failure report**, **service interval**.

### Roadmap (Q3 2026, owner: Helen Achebe)
| Item | Surface | Target | Status |
|---|---|---|---|
| Change to who gets pinged | Dispatch | 4.2 | Committed (shipped) |
| Availability Confidence (score next to stated availability) | Dispatch | 4.2 | Committed — driven by support escalations |
| Ping timeout tuning | Dispatch | 4.2 | Committed (shipped) |
| Requisition approval chains | Supply | 4.3 | Committed |
| Handler phone app | Supply | Q4 | Exploring |
| Shared cover between responders | Dispatch | Q4 | Exploring |

Committed items against a numbered release are locked; changes go through Product.

### People
| Name | Role | Notes |
|---|---|---|
| Helen Achebe | Director of Product | Your manager. Owns roadmap/commitments. |
| Marcus Oyelaran | Eng Manager, Dispatch | First stop when unsure. Can pull numbers. |
| Wen Li | Staff Eng, Dispatch | Built routing/ranking — the only real source on how it works. |
| Sofia Marino | Product Designer, Dispatch | Console + phone app. |
| Nadia Hoffmann | Support Lead | Sees complaint volume first. Worth a standing 15 min. |
| Ravi Menon | Data Analyst | The numbers; weekly acceptance reporting. Via #data. |
| Priya Raghunathan | Former PM, Dispatch | Departed 21 Aug 2026. Handover doc: `00-rook/company/notes/handoff-from-priya.docx`. |

### The 4.2 incident — current priority
4.2 rebalanced routing (proximity up, recent-acceptance down) and cut the offer timeout 90s→60s. Since then, acceptance is down and complaints are ~3x normal ("phone never goes off" / "offer expired before I could respond"). Prior PM's read — "seasonal, recovers by September" — **is disproven**: aggregate acceptance rate is bouncing back (77%→54%→73%), but the *spread* of callout volume across responders keeps widening (stdev 2.4 pre-release → 7.0 by 31 Aug). Four responders (Farlight, Meteor Mite, The Undertow, Vesper) are down to 0–1 callouts/week; four others (The Gale, Nightwell, Captain Vantage, Stormwrack) are up to 18–21/week. That's redistribution, not softness, and it's still worsening. *(Caveat: 2 tickets — Ironvale, Nightwell — don't match the CSV; confirm with Ravi whether `callout-history.csv` is real numbers or Marcus's rough 18 Aug pull.)*

**Root cause:** `history.py` scores a timed-out offer (`NO_ANSWER`) identically to an active decline, and the score never decays back to neutral (unresolved TODO since 2019). Harmless at 90s; at 60s, misses spike, tank a responder's score, drop their routing priority, and — with no decay — they can't recover without an offer they're no longer getting. Self-reinforcing both directions. Marcus flagged this exact gap in `#dispatch-team` on 14 Aug; never answered. Score is in-memory only (`_scores = {}`, `history.py`) — confirm with Marcus/Wen how it survives a deploy.

**Fix (both, in `history.py`/`offer.py`):** (1) smaller penalty for timeout than decline; (2) time-decay toward neutral. Reverting to 90s alone slows the spiral but doesn't fix it.

**Not to relitigate:** the proximity/acceptance rebalance itself — deliberate, long-requested; reverting just swaps one unhappy group for another.

**Next steps:** get the fix scoped with Marcus/Wen; keep Helen/Nadia anchored on the real cause so the rebalance doesn't get reverted by mistake; loop in support on the starved responders. Two more things from Priya's handover, neither currently tracked on the roadmap: reconcile which Q3 items got squeezed out of 4.2 (with Helen), and get the routing logic written down (today it only lives in Wen's head). Filter-persistence tickets are cosmetic noise — don't over-invest there.

### CSV deep-dive (module 3 findings)
The failure has two distinct phases, not one: (1) a **uniform shock** the release week — every one of the 16 responders' acceptance rate dropped double digits simultaneously (aggregate 77.9%→54.2%), consistent with the timeout cut alone; then (2) a **diverging redistribution** starting the following week that the timeout cut can't explain on its own, since it wasn't uniform — stdev of weekly volume per responder tripled (2.4→7.0) over the next three weeks and is still climbing.

Pre-4.2 acceptance rate only weakly predicts who crashes (r=0.285) — Vesper had one of the *highest* pre-4.2 rates (81.9%) yet crashed hardest of anyone (−81%). This suggests the proximity-weight increase (0.45→0.60) is doing real work in *who specifically* gets hurt, not just the scoring bug — but this CSV has no location/travel-time field to confirm it directly. Worth pulling routing/proximity logs from Wen to test this properly before finalizing the fix scope.

Tickets undercount the worst cases: of the 4 most-crashed responders, 2 (Vesper, Meteor Mite) never filed a single ticket — they only surface via the interviews. Don't use ticket volume as a proxy for who's hurting worst.

**Reporting script that tested well with Helen-style questions:** lead with the split, not the rate — *"4 of 16 responders are down 75–89% in the same 3 weeks that 7 others are up 30–49%"* — then explain why acceptance rate looks like it's recovering: the router is increasingly only asking people likely to say yes, so the rate can improve *because* the spiral is worsening, not despite it.

No documented testing/guardrail process exists anywhere in the source docs for this release (no staged rollout, canary, or rollback threshold found). The real miss was process, not test coverage: Marcus asked the exact right question in Slack the day after release (does the config distinguish decline from timeout?) and it was never answered.

### Code deep-dive (module 4 findings)
Read `00-rook/code/dispatch-routing/` line by line. **Marcus's unanswered 14 Aug Slack question is now answerable:** the 4.2 weight change was never scoped to new responders — `routing.py`'s `score()` (lines 38–42) is one formula with no branching, run fresh for every responder on every callout, so it applied instantly to everyone's *existing* history the moment it shipped, including responders who'd already been declining. Confirmed by both the code and the data (every one of the 16 responders' rate dropped the same release week — only possible if applied universally, not just to new cases). Worth actually posting this back to `#dispatch-team`.

**Exact 4.2 diff, confirmed line-by-line:** all of it is three constants in `config.py` (lines 9, 13, 14) — `OFFER_TIMEOUT_SECONDS` 90→60, `WEIGHT_PROXIMITY` 0.45→0.60, `WEIGHT_RECENT_ACCEPTANCE` 0.40→0.25. `routing.py`, `offer.py`, `history.py`, `availability.py` are byte-for-byte untouched. The scoring bug (timeout penalized same as decline, no decay) predates 4.2 entirely — 4.2 just changed the two numbers that made that pre-existing gap start actually hurting people.

**Confirmed: exactly one thing in this whole codebase adds points back** — `history.record_accepted()` (+0.08), fired only from `offer.py`'s `dispatch()` on a successful in-time accept. No decay, no reset, no second path anywhere. For someone already floored, the only way back in today is winning a ranking on proximity alone (60% of the score) against a lucky nearby incident, then answering inside 60s — pure chance, no self-correction. Stopgap advice for an affected responder in the meantime: stay marked available as much as possible, keep capability tags accurate, be ready to answer instantly — and get their handler to escalate directly rather than wait it out (Vesper and Meteor Mite waited and never recovered).

### Feedback: interviews vs. tickets
25 tickets (`00-rook/feedback/tickets/`): 16 quiet-spell, 6 missed/expired, 3 combined — 13 handler-filed (web console), 12 responder-filed (mobile, terse). 4 interviews (`00-rook/feedback/interviews/`, run by Sofia Marino for console redesign research — not originally about 4.2) independently corroborate the spiral: Kip watches two of his own responders side by side (Meteor Mite crashed, The Gale thriving) with nothing on screen explaining why — same divergence pattern as the CSV, spotted without knowing the routing bug existed. Ambrose told the same near-miss story twice, three weeks apart, unprompted both times — once as T-001, once in his interview.

Why both sources look different but aren't contradictory: tickets self-select for pain (only filed when something's wrong, so they skew toward the crashed responders); the interviews were sampled for UI research, not incident severity, so they show a truer mix of winners and losers.

Other UX asks surfaced in interviews, separate from the 4.2 bug — worth a product backlog, not urgent: console legibility/dark mode (3 of 4 interviews), filter-persistence trust, per-responder alert sound, handler-side notification when an offer comes in. Halloran also flagged Supply issues: requisition priority flag is ignored, failure reports get no feedback, catalog search is weak.

### Confidentiality
Responder cover identities are never stored or reconstructable in production. Don't design anything that assumes we can map a responder to a legal identity, and don't try to work out who anyone is.
