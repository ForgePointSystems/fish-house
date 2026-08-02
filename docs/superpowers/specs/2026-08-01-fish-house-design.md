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

### A separate Supabase project, ideally owned by the business

Not the `personal-automations` project, and ideally not a personal account at
all. Experiments must not be able to break production inventory, the business's
data should not sit inside a personal billing and backup scope, and if the
facility ever hires someone technical, handing over an organization they already
own is trivial where extracting their tables from someone else's project is not.

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
- Multi-facility support
- Replacing the Google Sheet mirror
