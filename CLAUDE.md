# fish-house

## What this project does
Inventory and lot traceability for a small fish processing facility, replacing a
hand-maintained Google Sheet. Tablet app in the processing room records pulls
against restaurant orders, decrements inventory, and prints a label.

## Stack
- Next.js (App Router) on Vercel — PWA, iPad-first
- Supabase (Postgres + Auth + Storage + Edge Functions)
- TypeScript
- Tailwind CSS
- Print agent: small on-site service, runtime TBD

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
Design phase. Architecture agreed in principle; blocked on
`docs/reference/open-questions.md` — the label printer, where orders live, and
whether lot numbers already exist in the sheet.
