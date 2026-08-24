# fish-house

## What this project does
Inventory and lot traceability for a small fish processing facility, replacing a
hand-maintained Google Sheet.

**v1 (current):** a real inventory ledger, an importer from the facility's
existing spreadsheet, and a UI to view on-hand stock and record receiving and
adjustments. No orders, no pulls-against-orders, no label printing yet — those
are designed and deferred, not forgotten. See
`docs/superpowers/specs/2026-08-01-fish-house-design.md` ("Scope reassessment")
for why.

## Stack
- Next.js (App Router) on Vercel — PWA, mobile-friendly
- Supabase (Postgres + Auth + Storage + Edge Functions)
- TypeScript
- Tailwind CSS
- Print agent: deferred to a later phase, not part of v1

## Handling real data
- **Never connect to the live Google Sheet.** Snapshots only — see README.
- `data/` is gitignored. Real inventory, pricing, and customer names live there.
  Never commit it, paste it into an issue, or send it to a third-party service.
- Sanitize before using any real row in a test fixture or example.

## Domain rules that are not negotiable
- **Inventory is an append-only ledger.** Every receipt, pull, and adjustment is
  an immutable row. On-hand is a derived view, never a stored number that gets
  overwritten. Do not add a mutable `quantity_on_hand` column.
- **Never block the floor.** A pull that exceeds recorded on-hand is recorded
  and flagged, not rejected. Negative on-hand is a reconciliation signal, not an
  error state.
- **No LLM in the inventory loop.** Arithmetic on weights is deterministic code.
  AI is allowed only at the edges (parsing emailed orders, flagging anomalies).
- **Lot lineage is load-bearing.** Traceability is the reason this is a database
  and not a spreadsheet. Never denormalize in a way that breaks the parent/child
  chain from receipt to shipment.
- **This is bespoke software for one facility. Build it specific.** No
  `facility_id`, no tenant discriminator, no configurability for hypothetical
  future customers — a second client would get their own repo and their own
  Supabase project. Generality here is pure cost. RLS is still used, for the
  ordinary reason: crew vs. owner permissions.
- **Use the crew's vocabulary in schema and UI** — see
  `docs/reference/glossary.md`. Where their term and ours differ, theirs wins.
- **Sheet sync is one-way, always.** Database to sheet, into a dedicated
  `FROM_APP` tab no human edits. Never write to their working tabs, never read
  the mirror back. Bidirectional sync is rejected, not deferred — spreadsheet
  rows have no stable identity and conflicts have no correct resolution. See the
  rollout section of the spec.
- **Rollback must always be "go back to the sheet."** That only works if the
  sheet stays current, which is why the mirror exists. It is an escape hatch, not
  a convenience.
- **Ledger rows carry a source** (app-entered, sheet-imported, count-adjustment)
  from the first migration, so the mirror and divergence report can tell them
  apart.
- Use the crew's vocabulary in schema and UI — see `docs/reference/glossary.md`.

## Project Structure
- `web/` — Next.js PWA (Vercel)
- `supabase/migrations/` — all DB schema changes
- `supabase/functions/` — edge functions
- `print-agent/` — on-site daemon, watches `print_jobs`, drives the label printer
- `docs/superpowers/specs/` — design docs
- `docs/superpowers/plans/` — implementation plans
- `docs/reference/` — glossary, open questions
- `data/` — sheet snapshots, gitignored

## Database Conventions
- Migrations live in `supabase/migrations/`
- Generate: `supabase db diff --use-migra -f migration_name`
- Never edit the DB directly in prod — always through migrations
- Apply to remote: `supabase db push`
- Generate types: `supabase gen types typescript --local > web/src/types/database.types.ts`

## Local Dev Commands
Nothing scaffolded yet. Fill in as the stack lands.

## Current State
Design phase, moving into implementation. v1 scope is inventory only (see
above). Blocking questions for the schema are now just open questions 12
(finfish-only vs. shellfish) and 13 (is Wanchese a second location) in
`docs/reference/open-questions.md` — the label printer and sales channel
questions no longer block anything since orders are deferred.
