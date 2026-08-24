# Fish House — Design

**Date:** 2026-08-01
**Status:** Draft. Architecture agreed in principle. **v1 narrowed to inventory
digitization on 2026-08-24** — see "Scope" below. Order entry, pulls-against-
orders, and label printing are real and still wanted, but deferred to a later
phase rather than designed now. Blocked on the open questions in
`docs/reference/open-questions.md`, most of which now concern only the
inventory slice.

## Purpose

Replace a fragile Google Sheet with a real database for a small fish processing
facility.

**v1's user-facing loop:** see current on-hand inventory on a screen instead of
a spreadsheet, log fish in as a lot when it's received, and record it leaving
inventory as an adjustment. No order linkage, no pull-specific workflow, no
label printing yet — those come once the inventory core is trusted. See
"Scope" below for why, and the deferred original loop for what's next.

## Context

The business runs entirely on Google Workspace. Inventory lives in one Google
Sheet maintained by hand. Labels are produced today by moving data into a second
sheet wired to a label printer.

The owner's own framing, which sets the priority:

> I've thought about trying to make it into an actual database. Probably the
> right place to start so there's a strong foundation.

The database is the foundation. The iPad app is the first thing built on it, not
the thing itself.

## Commercial intent

This is bespoke software for one facility, and it stays that way. It is not a
product, and it will not be sold to other processors.

The longer-term goal is a **services business**: building tooling tailored to
individual businesses, one repo and one deployment per client. This facility is
the first engagement. What carries forward to the next client is the stack, the
patterns, and the reputation — not this codebase.

The practical consequence is that this design should be **specific**, not
general. Model this facility's actual workflow. Do not hedge toward
configurability for hypothetical future customers who do not exist; that hedging
is pure cost in a per-client model, and it makes the software worse at the one
job it has.

## Scope reassessment (2026-08-11), resolved 2026-08-24

The first real documents arrived and were read — see
`docs/reference/sheet-findings.md`. They confirmed this is a **multi-channel
wholesale seafood distributor**, not a single-channel processing operation:
wholesale restaurant accounts (~150+, the largest single piece), retail chain
accounts with formal POs, a CSA-style subscription program, farmers markets,
out-of-state shipping, and a satellite Asheville/WNC operation, across finfish,
live shellfish, and dry goods.

The original scope ("processing team pulls fish for a restaurant order")
quietly assumed one channel, one product category, one location — none of
which are true. Rather than pick a channel/category/location boundary for the
*order* workflow, the resolution was to **not build the order workflow yet at
all**. v1 became inventory only: a real ledger, an importer, and a UI to see
and adjust it. That sidesteps the channel question entirely (open question 11)
and defers the shellfish-vs-finfish question (12) to just the inventory model,
where it's a much smaller decision than it would be for order fulfillment.

The architecture decisions below (append-only ledger, bespoke build, one-way
sync, no AI in the core loop, narrow-slice rollout) are all unaffected — if
anything they matter more now, since inventory-first is itself an application
of "narrow slice" thinking to the whole project, not just the rollout.

## Scope

**In scope for v1**

- Lot-level inventory as an append-only transaction ledger
- Receiving — incoming fish logged as lots with a lot number, parsed lot code
  fields, source/dock, and pickup date
- Adjustment — record product leaving inventory (sold, used, spoiled,
  correction) as a ledger entry, **not yet linked to a structured order**
- A UI: live on-hand by species (replaces the stale `Fish Qty` pivot),
  receiving screen, adjustment screen
- An importer for the `Inventory` tab, separating hand-typed facts from
  formula-derived columns (see `docs/reference/sheet-findings.md`) — this
  doubles as the phase-1 seed loader and the foundation of the eventual
  divergence checker
- One-way mirror of current on-hand back into a Google Sheet

**Out of scope for v1 — deferred, not abandoned**

- **Order entry, pulls-against-orders, and label printing.** This was the
  original first loop. It's still the goal — just sequenced after inventory is
  real and trusted, so it gets designed against actual data instead of
  guesses. Needs open questions 11 (channel) and 1 (printer) answered when it
  comes up; neither blocks v1 anymore.
- Purchasing, invoicing, accounting, payroll
- Anything customer-facing
- Formatted HACCP or regulatory reports. Traceability *data* is captured from
  day one; generating compliance paperwork from it comes later.
- Any LLM in the core inventory loop (see decisions below)

## Architecture

**v1:**

```
[PWA — tablet or desktop browser]
   view on-hand / receive lot / adjust
             │  HTTPS
             ▼
    [Supabase — Postgres]
      inventory_transaction (append-only)
      lots, species
      current_on_hand (view)
             │
             │ scheduled
             ▼
   [Google Sheet mirror]
      current on-hand, read-only
```

Two deployables, one repo: the web app (Vercel) and the database (Supabase).
No print agent, no `print_jobs`, no `orders` table yet — see "Deferred" below.

**Future phase, once orders are tackled** (not designed yet, kept here so the
v1 schema doesn't accidentally close off the path):

```
             ┌──────────────────────┐
             │  orders, print_jobs   │
             │  added to the schema  │
             └──────────┬────────────┘
                         │ Realtime/poll
                         ▼
              [print agent — on-site]
                 raw label commands
                         │
                         ▼
                 [label printer]
```

A pull would write a row to `print_jobs`; an always-on on-site machine (an
existing PC or a Raspberry Pi) watches it and drives the printer directly, so
the tablet never talks to the printer. That reasoning doesn't change — it's
just not being built yet.

## Key decisions

### Progressive web app, not a native iOS app

Added to the iPad home screen, a PWA runs fullscreen with no browser chrome —
indistinguishable from an installed app to the crew, which is all "installed on
our iPad" requires. It avoids the Apple developer account, App Store review, and
the update cycle: a fix pushed mid-shift is live on next launch.

The cost is Web Bluetooth, which iOS Safari does not support. If the facility
later wants a Bluetooth scale pushing weights automatically rather than the crew
typing them, that single feature would force a native wrapper. Camera-based
barcode scanning works fine on iOS, so scanning is not affected.

### A dedicated Supabase project for this client

Not the `personal-automations` project — experiments must not be able to break
production inventory, and the two have nothing to do with each other.

One project per client, always. It is the cleanest possible blast radius: a
mistake here cannot touch another client's data, because there is no shared
database to make a mistake in. It also makes an eventual handoff or exit trivial
— the client's entire system is one project and one repo.

### Single tenant, and genuinely single tenant

**No `facility_id`, no tenant discriminator, no configurability for
hypothetical customers.** This database serves one facility. A tenant column
whose value is always the same is not insurance, it is a filter on every query
and a lie about what the system is.

If a second processor ever wants something similar, they get their own repo,
their own Supabase project, and a design that fits *their* workflow — which is
the entire premise of the services model. Nothing about this schema needs to
anticipate them.

RLS is still used, for the ordinary reason: distinguishing what a processing
crew member can see and do from what the owner can. That is a real requirement
today, not a hedge.

Where something is genuinely variable *within this facility* — the species they
handle, their product forms, their storage locations — it is a table because
that is correct schema design, not because a future customer might differ.

### Inventory is an append-only ledger, not a quantity column

Every receipt, pull, and adjustment is an immutable row. On-hand is a view
derived from the ledger, never a stored number that gets overwritten.

This is the difference between a real foundation and a spreadsheet with extra
steps. A cell holding `450` that someone overwrites with `380` has destroyed the
only record that 70 lbs moved. A ledger cannot lose that. It also means two
people pulling at the same time cannot clobber each other — concurrent inserts
are a non-event where concurrent cell edits are a corruption — and it makes lot
lineage walkable, which is what recall traceability actually requires.

### The Google Sheet stays alive as a one-way mirror

Supabase becomes the system of record, but a scheduled job writes current state
back out to a Google Sheet. Reports, habits, and other people's workflows hang
off that sheet in ways neither we nor the owner can fully enumerate. Mirroring
keeps all of it working, preserves the owner's muscle memory, and means a bad
week for the app is not a bad week for the business.

The mirror is one-way and read-only by convention. Once the app is trusted, the
mirror can be dropped.

### Printing goes through a local agent, not the iPad *(deferred with the rest of the order/pull/label workflow — decision kept for when it's built)*

A pull writes a row to `print_jobs`. An always-on machine in the building — an
existing PC or a Raspberry Pi — watches that table and pushes raw label commands
to the printer over the network. The iPad never talks to the printer.

Printing from iOS to a label printer directly is awkward, and coupling the two
would make the app's reliability depend on the tablet being awake, in range, and
on the right network. Decoupling also gives reprints for free with a full
history, which will be needed the first time a label jams or gets soaked.

### No AI in the core loop

Subtracting weight from a lot is deterministic bookkeeping. A model must never
be in the path of deciding how much fish left the freezer — that is what makes
the numbers trustworthy on the floor.

AI earns its way in later at the edges, where it is doing extraction rather than
arithmetic: parsing emailed orders into order records, flagging anomalous yields
for a human to look at.

### The app never blocks the floor

If the system says 12 lbs are on hand and there are really 18, the pull is
recorded and the discrepancy flagged — the app does not refuse the transaction.

An app that halts the processing line because its own numbers were stale gets
abandoned within a week, and the spreadsheet comes back. Negative on-hand is a
signal to reconcile, not an error state to enforce.

### Oldest lot first, with override *(deferred with the rest of the order/pull/label workflow — decision kept for when it's built)*

When several lots of one species are on hand, a pull draws from the oldest by
default so the common case is a single tap, with a manual override for when the
crew knows better. The lot chosen determines the lot number printed on the
label, so this cannot be left implicit.

## Consequences worth naming

**Receiving is in v1 scope regardless of pulls or labels.** The original
reasoning was "labels need a lot number, so intake must be captured" — that
reasoning no longer applies directly, since labels are deferred. But receiving
is in scope anyway: without it there's nothing to seed the ledger with beyond
the one-time import, and no way to add new lots as fish actually arrives. It
was always going to be needed; it just doesn't need justifying by labels
anymore.

**The floor is the real risk, not the database.** The sheet survives because it
is forgiving and already known. Whatever replaces it has to be faster to enter
data into than the sheet, on a tablet, with cold wet hands, possibly on bad wifi
near a freezer. That is a client-side offline-queue problem, and it is where
projects like this usually die.

## Rollout and migration

**Status: sketch.** The shape is settled; the specifics need fleshing out once
we have the sheet and know how the facility actually works.

### Sync is one-way. Bidirectional sync is not an option.

Not "deferred" — rejected. Two-way sync between the sheet and the database
cannot be made correct:

- **Spreadsheet rows have no stable identity.** There is no primary key. Rows
  are sorted, inserted, dragged, and cleared, so a row is identified only by
  position or by a fuzzy composite of its values. Any matching logic will
  eventually pair the wrong rows and corrupt inventory silently.
- **Conflicts have no correct resolution.** If a cell is edited to `380` while
  the app records a 70 lb pull, last-write-wins destroys one of them, and nobody
  learns about it until a count fails to reconcile. A spreadsheet edit carries no
  intent, so "corrected a typo" and "recorded a real movement" are
  indistinguishable.
- **It loops.** Database writes to the sheet, a sheet trigger fires, it writes
  back to the database.

### Authority is partitioned by scope, not by time

The way to avoid sync conflicts entirely is to give every piece of data exactly
one home at any moment. Rather than both systems holding everything and
reconciling, the business is split into slices and moved one slice at a time.

The slice must have a boundary that is **physically real to the crew** — one
species, one freezer, one shift — so nobody has to remember a rule about where
something lives.

For v1 (inventory only, no orders yet), the natural slice is **species or
product category** — this fits the ledger directly. When the order/pull/label
workflow eventually gets built, a different boundary becomes available:
**route or driver**. The order/dispatch document is already organized that
way, and each route reads like it's maintained by one person — so piloting
with one route only touches what that person currently writes by hand, leaving
every other route's workflow completely undisturbed. Worth raising with the
owner directly when that phase starts: "let's start with Tuesday's south
route" rather than "let's start with salmon." Noted here so the idea isn't
lost between now and then.

### Phases

1. **Read-only shadow.** Database seeded from a snapshot. The app displays
   inventory but is authoritative for nothing. The purpose is finding out where
   our model of a lot disagrees with theirs, cheaply.
2. **Dual entry on one slice.** Pulls of the chosen slice are entered in both
   systems for roughly two weeks. Nothing syncs. **The sheet remains the system
   of record** — when they disagree, the sheet wins and the app gets fixed. The
   double entry is the cost of the highest-confidence validation available, which
   is why the slice is narrow and the window is short.
3. **Flip authority for that slice.** The database becomes truth for the slice. A
   scheduled job writes derived state one-way into a dedicated `FROM_APP` tab
   that no human edits. Existing reports keep working; the app never reads that
   tab back.
4. **Widen slice by slice** until nothing is left outside the app, then freeze
   the sheet as an archive.

### The divergence report

A nightly job pulls the sheet, diffs it against the database, and sends a short
summary of what disagrees.

This is the highest-value thing built during the trial. It converts "I hope this
is working" into evidence, it catches problems while the sheet is still there to
fall back on, and it is the artifact that earns the owner's go-ahead for the
next phase.

### Cutover attaches to their physical count

The facility already counts inventory periodically. That is the natural resync
point: after a count, both systems are set to observed truth and divergence
resets to zero. Do not invent a reconciliation ritual — attach to the existing
one.

### The rollback rule

**At every phase, the answer to "the app is broken" must be "go back to the
sheet," and that only works if the sheet is still current.**

The one-way mirror in phase 3 exists for this reason. It is not a convenience
feature, it is the escape hatch.

### Schema consequence

Ledger rows carry a **source** — app-entered, sheet-imported, count-adjustment —
so the mirror and the divergence report can distinguish what the app knows from
what was brought in from outside. Needed from the first migration.

## Open questions

Tracked in `docs/reference/open-questions.md`. With v1 narrowed to inventory
only, the questions that actually block schema work now are: whether v1
covers finfish only or also shellfish (open question 12), and whether Wanchese
is a second physical location that needs its own on-hand scope or just cold
storage tied to the main facility (open question 13). The label printer
(question 1) and which sales channel orders belong to (question 11) no longer
block anything — they matter once the order/pull/label phase starts.

## Deferred (explicitly not in v1)

- **Order entry, pulls-against-orders, label printing, and the print agent.**
  The original first loop. Still wanted — sequenced after inventory is real,
  so it can be designed against actual data instead of assumptions. See
  "Scope reassessment" above.
- Offline queue on the tablet. Needed eventually; deferred until we know how bad
  the wifi actually is in the processing room.
- Bluetooth scale integration (blocked by iOS, see above)
- Yield analysis and margin reporting — the ledger makes these possible later
  without schema changes
- Formatted HACCP and recall reports
- Replacing the Google Sheet mirror
- Multi-tenancy, signup, billing, tenant admin — permanently out of scope, not
  deferred. A second client means a second repo.
- Any claim that the system produces compliant HACCP or regulatory records.
  Capturing traceability data is v1; asserting regulatory sufficiency is a legal
  posture, not a feature, and must not be claimed until it has been reviewed.
