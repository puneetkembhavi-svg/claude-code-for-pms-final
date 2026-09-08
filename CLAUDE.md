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

New hire: PM for **Rook Dispatch**, starting now. Compiled from `00-rook/company/` on 8 Sept 2026.

### What Rook does
Coordination and provisioning software for the protective-response sector — publicly framed as logistics/workforce-coordination for emergency services. Real customers are independently-operating responders and the handlers/quartermasters who support them. Two product surfaces:

- **Rook Dispatch** — responder coordination: availability, proximity, callout routing, acceptance. Used by handlers (web console) and responders (mobile). *This is your product.*
- **Rook Supply** — gear provisioning: requisitions, maintenance, failure reports. Used by handlers and quartermasters.

Ships monthly on a release train (4.x numbering). Current release: **4.2** (shipped 12 Aug 2026).

### How Dispatch works
Incident arrives (handler entry or intake system) → Dispatch ranks available responders (**routing priority**) → top-ranked responder gets a **callout offer** on mobile → accept, or decline/timeout moves it to the next responder → acceptance marks the responder engaged.

Routing priority inputs: proximity (travel-time estimate), current availability, capability match, recent acceptance history. Declining/timing out lowers a responder's recent-acceptance component, which lowers their routing priority until it recovers. Routing config ships with releases — not a runtime setting handlers can adjust.

**Headline metrics:** callout acceptance rate (weekly, aggregate), time-to-accept (median seconds), coverage gap (incidents with no capability-matched responder available).

### Vocabulary
- **Responder** — independent field operator, not a Rook employee. Exists in our systems only as capability tags + availability; never a legal identity.
- **Cover identity** — a responder's public persona. Rook holds no mapping to a legal identity — don't design as if we do. See Security Policy 4.1 before touching responder records.
- **Handler** — manages a responder or small group: availability, gear, readiness. Usually the one actually using the product.
- **Quartermaster** — owns equipment stock/approvals (Supply-side).
- **Callout / callout offer / callout timeout** — a request for a responder; the offer to one specific responder; how long that offer stays live before moving on (currently 60s, cut from 90s in 4.2).
- **Decline vs. timeout** — distinct in the data; both send the callout onward.
- **Coverage gap** — nobody available had the right capability tags (different from low acceptance — nobody *could* go, vs. nobody *did*).
- **Capability tag** — flight, structural-entry, hazmat-tolerant, cold-weather, aquatic, crowd-management, de-escalation.
- **Responder Availability Record** — shared record of responder availability; **written by Dispatch, read-only for Supply** (Supply uses it to schedule maintenance around callout load).
- **Mutual aid** — cross-region coverage between responders. Not built yet; Q4 exploration.
- Supply terms you'll hear on shared calls: **requisition**, **field failure report**, **service interval**.

### Where things stand (as of hire date)
4.2 (12 Aug) rebalanced routing to weight proximity more heavily vs. recent acceptance history — a long-requested change for responders in wide geographies, sat on for three quarters. Also cut callout timeout 90s→60s, and shipped filter persistence + 3 defect fixes.

Since the release, acceptance is down and complaint volume is ~3x normal, split into two distinct issues: (1) "phone never goes off" and (2) "offer expired before I could respond." Support (Nadia) has been tracking the split since 18 Aug. Prior PM's read was "mostly seasonal, will recover in September" — **investigated below, and that read does not hold up.**

Explicitly **not** to relitigate: reverting the 4.2 routing change itself (proximity/acceptance rebalance) — it was a deliberate, long-requested fix and reverting trades one unhappy group for another. The timeout cut is a separate, more defensible thing to revisit (see below).

### 4.2 root-cause investigation (findings, 9 Sept 2026)

**Not seasonal.** Aggregate acceptance rate did crater the week 4.2 shipped (77%→54%) and has been climbing back since (66%, 73%) — this is what makes the seasonal read look plausible. But the *spread* of callout volume across responders has been widening every week since release, not recovering: weekly std. dev. per responder went 2.4 (stable for 6 weeks pre-4.2) → 4.7 → 6.4 → 7.0. By the week of 31 Aug, four responders (Farlight, Meteor Mite, The Undertow, Vesper) are down to 0–1 callouts/week from a normal 10–14, while four others (The Gale, Nightwell, Captain Vantage, Stormwrack) are up to 18–21/week from a normal 12–15. Seasonal softness would depress everyone evenly; this is redistribution, concentrating callouts onto a shrinking pool of responders — and it's actively getting worse, not settling down. (Caveat: 2 tickets — Ironvale, Nightwell — don't match this CSV pattern; worth confirming with Ravi whether `00-rook/data/callout-history.csv` is the real weekly numbers or the rough pull Marcus mentioned pulling by hand on 18 Aug, before leaning on it further.)

**Mechanism:** a pre-existing gap in `history.py` (see `00-rook/code/dispatch-routing/history.py`) scores a timed-out offer (`NO_ANSWER`) identically to an active decline — both call `record_declined()`, same penalty. There's also no time-decay: a responder's recent-acceptance score only moves via accept/decline, never drifts back toward neutral on its own (unresolved TODO in that file since 2019). This was harmless at a 90s timeout; cutting it to 60s in 4.2 made misses more common, which triggers the same penalty as a refusal, which lowers routing priority, which means fewer future offers, which — with no decay — the responder can't recover from without an offer they're no longer getting. Self-reinforcing in both directions: responders already winning climb further too. Marcus asked whether the config distinguishes decline from timeout in `#dispatch-team` on 14 Aug; nobody ever answered it.

**Fix options (either helps, neither alone is complete — do both):**
1. Give `NO_ANSWER` a smaller (or zero) penalty than an active `DECLINE` in `offer.py`/`history.py`.
2. Add time-based decay in `history.py` so score drifts back toward `NEUTRAL_SCORE` (0.5) the longer a responder goes without an offer.

Reverting the timeout to 90s alone would reduce how often misses happen and slow the spiral, but wouldn't fix it — the scoring bug would still ratchet down anyone having a bad week. Not a substitute for the code fix above.

**Score storage detail:** the recent-acceptance score lives only in an in-memory dict (`_scores = {}` in `history.py`) — not persisted elsewhere, worth confirming with Marcus/Wen how that survives a deploy or restart before relying on it for anything.

**My immediate next steps on this:** (1) take the fix to Marcus/Wen and get it scoped into a release — this is compounding weekly; (2) make sure Helen and Nadia see the divergence data so nobody reverts the 4.2 proximity/acceptance rebalance itself as an overreaction to the wrong root cause; (3) loop in support/handlers on what's actually happening to the starved responders (Farlight, Meteor Mite, The Undertow, Vesper) so they're not left thinking their accounts are broken.

**Note:** neither "reconcile squeezed-out Q3 items with Helen" nor "write the routing spec" is actually tracked on the Q3 roadmap today — both are informal follow-ups from Priya's handover, not committed work. Worth getting them onto the roadmap properly if they're going to happen.

Other open threads from handover:
- No written spec exists for how routing/"who gets pinged" actually works — Wen Li built it, it lives in her head. Worth writing down.
- Some Q3-committed items got squeezed out of 4.2 scope; unclear which are still committed. Needs a conversation with Helen.
- Console filter-persistence change will generate tickets but is cosmetic/noise — don't over-invest there.

### Roadmap (Q3 2026, owner: Helen Achebe)
| Item | Surface | Target | Status |
|---|---|---|---|
| Change to who gets pinged | Dispatch | 4.2 | Committed (shipped) |
| Availability Confidence (confidence score next to stated availability) | Dispatch | 4.2 | Committed — driven by support escalations |
| Ping timeout tuning | Dispatch | 4.2 | Committed (shipped) |
| Requisition approval chains | Supply | 4.3 | Committed |
| Handler phone app | Supply | Q4 | Exploring |
| Shared cover between responders | Dispatch | Q4 | Exploring |

Committed items against a numbered release are locked; changes go through Product.

### People
| Name | Role | Surface | Notes |
|---|---|---|---|
| Helen Achebe | Director of Product | Dispatch & Supply | Your manager. Owns roadmap/commitments. |
| Marcus Oyelaran | Engineering Manager | Dispatch | Runs Dispatch eng. Straight talker, first stop when unsure. Can pull numbers. |
| Wen Li | Staff Engineer | Dispatch | Built routing/ranking. The only real source on how it works. Was away 14–24 Aug. |
| Sofia Marino | Product Designer | Dispatch | Console + phone app. |
| Nadia Hoffmann | Support Lead | Dispatch & Supply | Sees complaint volume first. Worth a standing 15 min. |
| Ravi Menon | Data Analyst | Dispatch & Supply | The numbers; weekly acceptance reporting. Requests via #data. |
| Priya Raghunathan | (former PM, Dispatch) | — | Departed 21 Aug 2026. Left a handover doc — see `00-rook/company/notes/handoff-from-priya.docx`. |

### Confidentiality
Responder cover identities are never stored or reconstructable in production — capability tags, availability, and callout history only. Don't design anything that assumes we can map a responder to a legal identity, and don't try to work out who anyone is.
