---
name: review-checklist
description: Run a fixed, personal four-point check against a product brief or one-pager before it goes further — checks that the brief names an owner, states how success will be measured, keeps its scope bounded (matches at the end what it promised at the start), and explains the problem before proposing the fix. Use this whenever the user asks to review, check, or run their checklist against a brief, one-pager, PRD, or proposal — including phrases like "check this brief," "run review-checklist," "does this brief pass," "review this before it goes to [someone]," or when pointed at a file in a briefs/ folder. Works on a single file or a whole directory of briefs in one pass. Always use the exact four checks and report format below — this skill exists so the same four things get checked the same way every time, not re-derived per document.
---

# Review checklist

A personal QA pass the user runs on every brief before it goes further — to
engineering, to a director, to anyone who'll act on it. The value of this
skill is *consistency*: the same four checks, checked the same way, reported
in the same format, every single time. Don't improvise new checks, don't
skip one because the document "seems fine," and don't soften the format
because a brief is short or informal. A brief that's easy to read is still
worth checking — the checks exist because gaps hide even in brief, well-
written documents.

## The four checks

Run these in order, and check them independently — a brief can pass three
and fail one.

### 1. Owner named
Does the document assign the work to a specific person or team who is
actually responsible for it going forward — not just mention people who
were consulted, and not just a promise that someone will own it eventually?

- **Passes**: an explicit owner ("Marcus's team builds it," "Halloran's
  team, Supply") — a name or named team taking responsibility.
- **Fails**: the document only *mentions* people (design input needed from
  X, engineering will need to scope Y) without anyone actually holding it,
  or explicitly defers ownership to the future ("flagging this for whoever
  picks it up next quarter... nobody's picked it up yet"). Flagging a gap
  for a future owner is not the same as naming one — that's the exact
  distinction this check exists to catch.

### 2. Success measure
Does the document say how anyone would know it worked — an observable
signal or metric, not just a description of what the thing does or where
it shows up?

- **Passes**: a stated measure, even an imperfect one. "Fewer support
  tickets mentioning X" or "that number should be zero" both pass, even if
  the document admits there's no baseline yet — stating *what* you'd watch
  is what this check is for, not whether the measurement infrastructure
  already exists. Don't fail a brief just for lacking a baseline if it
  correctly names the metric.
- **Fails**: the document describes the feature and its mechanics in full
  (what gets built, where it appears) but never says what "working" would
  look like from the outside.

### 3. Scope stays bounded
Does the scope described at the end of the document match what was framed
at the opening — or did it quietly grow along the way?

- **Passes**: the document either stays to one ask throughout, or
  explicitly draws a boundary ("just the three things above," "not a
  general audit log — just the one decision that currently leaves no
  trace").
- **Fails**: the document opens with one bounded ask (sometimes literally
  saying "that's the whole ask") and then, in a later paragraph, adds more
  — a related feature, a dashboard, a visibility change, a bigger
  replacement project — without ever re-scoping the opening framing to
  match. The tell is a document arguing with its own first paragraph.

### 4. Problem stated before fix
Does the document explain the problem *before* it proposes the solution —
problem-first ordering, not solution-first?

- **Passes**: a "Problem" (or equivalent) section precedes the "Proposal"
  section, and that section actually describes what's going wrong for
  someone, not just sets up the pitch.
- **Fails**: the document opens with the proposal — what will be built and
  what it does — and the actual problem only shows up later, or only as
  justification bolted onto the pitch after the fact. Order matters here
  more than content: a document can eventually explain the problem well
  and still fail this check if the fix came first.

## How to run it

1. Read the target file (or every file in the target directory) in full.
2. For each document, evaluate all four checks independently. Use **yes**
   when a check clearly passes, **NO** when it clearly fails, and
   **partial** only when a check is genuinely attempted but incomplete in
   a way that isn't a clean pass or fail (e.g. an owner is named but only
   as "TBD, pending someone being assigned" — reserve this for real
   ambiguity, not as a hedge; most real briefs will land on yes or NO).
3. Only flagged checks (NO or partial) get a reason. Passing checks don't
   need justification — that's what keeps the report scannable.
4. Every reason should point at *something specific in the document* — a
   short quoted phrase or a precise description of what's missing or where
   the scope grows — not a generic restatement of the rule.

## Report format

Use this exact structure for every document — pad each check label with
periods so the four results line up in one column (pad to 30 characters
before the result, roughly matching the line width below):

```
<filename or document title>
  owner named ................. <yes|NO — flagged|partial — flagged>
  success measure .............. <yes|NO — flagged|partial — flagged>
  scope stays bounded .......... <yes|NO — flagged|partial — flagged>
  problem stated before fix .... <yes|NO — flagged|partial — flagged>
  -> <N> flag(s): <one line per flag, plain sentence, pointing at the
     specific evidence in the document>
```

If nothing is flagged, the last line reads `-> No flags.` instead.

When checking more than one document in a single run (a directory of
briefs), print one block per document in the order the files were given
or found, then end with one summary line across all of them:

```
<N> briefs checked, <M> flagged<, distribution if it's not uniform>.
```

("4 briefs checked, 4 flagged, 1 flag each." — or "5 briefs checked, 2
flagged." if most passed clean — adapt the trailing clause to whatever
actually happened; don't force it into a template that doesn't fit.)

For a single document, skip the cross-document summary line — the
per-document block is the whole report.

### Worked example

For a document that opens with a bounded ask, closes by deferring
ownership to an unnamed future owner, and states a success measure with
no baseline yet:

```
bulk-callout.txt
  owner named ................. NO — flagged
  success measure .............. yes
  scope stays bounded .......... yes
  problem stated before fix .... yes
  -> 1 flag: no owner named. Ends on "nobody's picked it up yet" —
     flagging it for a future owner isn't the same as naming one.
```

## Notes

- This skill is for *reviewing* a brief, not rewriting it. Report what's
  there — don't suggest fixes, don't rewrite the document, don't editorialize
  beyond the one-line reason for each flag. If the user wants help fixing
  a flagged brief, that's a separate ask.
- If a target isn't a brief/one-pager at all (e.g. raw code, a spreadsheet),
  say so plainly rather than forcing the four checks onto it.
- These four checks are fixed by design — don't add a fifth check or drop
  one because it doesn't seem to apply. If a document genuinely has no
  scope to speak of (a single-paragraph note), still run all four checks;
  say so in the reason line rather than skipping the check.
