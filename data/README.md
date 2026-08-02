# data/

Snapshots of the owner's Google Sheet, and any other real business data.

**Everything in here is gitignored except this file.** It contains a real
business's inventory, pricing, and customer names. Do not commit it, paste it
into an issue or a chat, or send it to a third-party service.

## Getting a snapshot

Point-in-time exports only — never a live connection to the working sheet.

```bash
curl -L "https://docs.google.com/spreadsheets/d/FILE_ID/export?format=xlsx" -o data/samples/inventory-2026-08-01.xlsx
```

`FILE_ID` is the string between `/d/` and `/edit` in the sheet URL. A GET
against the export endpoint renders a copy server-side and cannot mutate the
source.

If it returns a permission error, the owner has unchecked *"Viewers and
commenters can see the option to download, print, and copy"* in the share
dialog's gear menu. He needs to re-enable it.

## Conventions

- Name files `inventory-YYYY-MM-DD.xlsx` — the date is the snapshot date
- Ask for snapshots taken at a quiet moment (end of day, not mid-receiving), so
  the numbers are internally consistent rather than half-updated
- Test fixtures derived from this data get sanitized first: fake restaurant
  names, scrambled prices
