# Fish House — Design

**Date:** 2026-08-01
**Status:** Draft. Architecture agreed in principle. Blocked on the open
questions in `docs/reference/open-questions.md` before a schema can be written.

## Purpose

Replace a fragile Google Sheet with a real database for a small fish processing
facility, and put a tablet app in the processing room on top of it.

The first user-facing loop: the processing team pulls fish for a restaurant
order, records the weight pulled, inventory decrements, and a label prints
identifying which order the fish is for.

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

## Scope reassessment (2026-08-11)

The first real documents arrived and were read — see
`docs/reference/sheet-findings.md`. They confirm this is a **multi-channel
wholesale seafood distributor**, not a single-channel processing operation:
wholesale restaurant accounts (~150+, the largest single piece), retail chain
accounts with formal POs, a CSA-style subscription program, farmers markets,
out-of-state shipping, and a satellite Asheville/WNC operation, across finfish,
live shellfish, and dry goods.

**Nothing below in this document is invalidated by that** — the architecture
decisions (append-only ledger, bespoke build, one-way sync, no AI in the core
loop, narrow-slice rollout) all still hold, and arguably matter more now. What
changes is that **v1's boundary must be a deliberate choice, not the implicit
one this document originally made** ("processing team pulls fish for a
restaurant order" quietly assumed one channel, one product category, one
location). See open questions 11–13 for the specific choices pending.

The scope section below should be read as **pre-reassessment** — a reasonable
first cut, now due for a revisit once the channel/category/location boundary is
picked.

## Scope

**In scope for v1**

- Lot-level inventory as an append-only transaction ledger
- Receiving — incoming fish logged as lots with a lot number
- Pull — record weight pulled against an order, decrement on hand
- Label printing triggered by a pull: restaurant, date, species, lot number
- One-way mirror of current on-hand back into a Google Sheet

**Out of scope for v1**

- Purchasing, invoicing, accounting, payroll
- Anything customer-facing
- Formatted HACCP or regulatory reports. Traceability *data* is captured from
  day one; generating compliance paperwork from it comes later.
- Any LLM in the core inventory loop (see decisions below)

## Architecture

```
[iPad — PWA, processing room]
   receive lot / pull for order
             │  HTTPS
             ▼
    [Supabase — Postgres]
      inventory_transaction (append-only)
      lots, orders, species
      current_on_hand (view)
      print_jobs
             │                    │
             │ Realtime/poll      │ scheduled
             ▼                    ▼
   [print agent — on-site]   [Google Sheet mirror]
      raw label commands         current on-hand,
             │                   read-only
             ▼
      [label printer]
```

Three deployables, one repo: the web app (Vercel), the database and edge
functions (Supabase), and a small always-on print agent inside the building.

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

### Printing goes through a local agent, not the iPad

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

### Oldest lot first, with override

When several lots of one species are on hand, a pull draws from the oldest by
default so the common case is a single tap, with a manual override for when the
crew knows better. The lot chosen determines the lot number printed on the
label, so this cannot be left implicit.

## Consequences worth naming

**Lot numbers on labels mean receiving must be captured.** The app cannot print
a lot number it was never told. An intake screen is therefore in v1 scope even
though the owner's description started at the pull step — unless lot numbers
already exist in the current sheet and can be seeded from it.

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

Tracked in `docs/reference/open-questions.md`. The blocking ones are the label
printer's make, model, and connection; where orders live today; and whether lot
numbers are already tracked in the sheet.

## Deferred (explicitly not in v1)

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
