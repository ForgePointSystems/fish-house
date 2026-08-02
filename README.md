# fish-house

Inventory and lot traceability for a small fish processing facility, replacing a
hand-maintained Google Sheet.

The first user-facing piece is a tablet app in the processing room: the crew
records fish pulled for a restaurant order, inventory decrements, and a label
prints identifying the order.

**Status:** design. Nothing is built yet. Start with
[the design doc](docs/superpowers/specs/2026-08-01-fish-house-design.md), then
[the open questions](docs/reference/open-questions.md).

## Handling the owner's data

**Never connect anything to the live Google Sheet.** Work from a point-in-time
snapshot:

```bash
curl -L "https://docs.google.com/spreadsheets/d/FILE_ID/export?format=xlsx" -o data/samples/inventory-YYYY-MM-DD.xlsx
```

A GET against the export endpoint cannot mutate the source. Equivalent, and
fine: `File → Download → Microsoft Excel (.xlsx)` from a viewer link.

**`data/` is gitignored and stays that way.** It holds a real business's
inventory, pricing, and customer names. Nothing in it gets committed, pasted
into an issue, or sent to a third-party service.

## Layout

```
web/          Next.js PWA for the iPad — Vercel
supabase/     Postgres schema, migrations, edge functions
print-agent/  on-site daemon: print_jobs table -> label printer
docs/         design docs, specs, plans, domain reference
data/         sheet snapshots (gitignored, never committed)
```

Three deployables in one repo because they ship together and share a schema.

## Getting started

Nothing to run yet. When the stack lands, commands go in
[CLAUDE.md](CLAUDE.md).
