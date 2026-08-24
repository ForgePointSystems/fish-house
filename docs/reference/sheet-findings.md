# Findings from the first real documents

**Date received:** 2026-08-03 (snapshot date). **Read:** 2026-08-11.

Source files (gitignored, not in this repo — see `data/README.md`):
`data/samples/inventory-2026-08-03.xlsx`,
`data/samples/order-dispatch-doc-2026-08-03.docx`.

This is a factual record of what the documents contain, kept separate from
`open-questions.md` (decisions still needed) and `glossary.md` (agreed
vocabulary), so the raw evidence stays available when those get revised.

## The inventory workbook — 8 tabs

| Tab | Rows | What it is |
|---|---|---|
| `Inventory` | ~687 | Master lot ledger. One row per lot: lbs remaining, species code, **Histamine Lot Code**, landed-at (dock/fisherman), pickup date, yield percentages by cut (skin-on/off/skin-gutted), price per cut, potential revenue by sales channel (wholesale, Asheville, HQ market, Saturday retail). |
| `Fish Qty` | ~710 | Pivot-style summary: species code, total lbs remaining, total skin-on yield, total skin-off yield. Reads as a derived/computed view, not a source of entry. |
| `OYP` | ~1031 | Oyster planning — weekly committed vs. held quantities, broken out by product type (Half Shell, Steamers, Clams) and by route/day (MON Central, MON OBX, THURSDAY OBX, etc). |
| `Frozen Fillets LS - 2026 new` | ~97 | Packs put up / sold / remaining for a specific frozen-fillet product line, with per-species pricing. |
| `BULK FRZN SMOKE` | ~93 | Same pattern as above, for bulk-frozen and smoked product. |
| `FF LOG` | ~344 | Processing/costing log: date purchased, date processed, species code, purchase price, treatment, source (vendor name), start weight, finish weight, **true yield**, packages produced, cost per package, estimated wholesale price. This is the yield-and-costing ledger. |
| `OY PLAN` | ~1026 | Oyster commitment planning by day and by named buyer/contact — similar in spirit to `OYP`, unclear yet if duplicate or sequential (weekly rebuild?). |
| `Wanchese Frozen Inv` | ~56 | Same packs put up/sold/remaining pattern as the frozen fillet tabs, but for a named location, Wanchese — see open question 13. |

**Every finfish lot carries a Histamine Lot Code.** Format:
`{SOURCE}-{SPECIES}-{MMDDYY}`, e.g. `PDL-ALM-080326` (Peddler dock, Almaco
jack, landed 8/3/26), `SLB-ALM-080326` (Selby dock, same species/date).
"Landed at" names a specific dock or fisherman per lot, e.g. "Peddler (Holden
Beach, NC)", "Selby (Hampstead, NC)" — this is genuine catch provenance, not
just a receiving date.

**Yield is already tracked**, at least in `FF LOG` and the `Inventory` tab's
percentage columns — this is not something v1 introduces, it's something v1
needs to preserve.

## The order/dispatch document

One continuous Word document per week ("ORDERS - current week"), structured as:

```
[Day] - [Date]
  [Route/driver header, e.g. "THU CENTRAL ROUTE - MIKE"]
    RESTAURANT NAME [invoice#] inv: <free-text quantities and species codes>
    RESTAURANT NAME inv: NO                    (no order this stop)
    > delivery instructions, gate codes, contact names, addresses
  [next route]
  ...
EOD [DAY] INVENTORY
  <flat list: quantity + species code + optional lot/date>
[next day]
```

Confirmed structural elements:

- **~150+ named wholesale accounts** recur week to week, organized by delivery
  day and route, each with a named driver (Michael, Tammy, Bill, Ryan & JV,
  Mike, Greg, Tabitha) and sometimes a specific vehicle ("drive MB3
  RALEIGH/CARY", "BIG TRANSIT").
- **Invoice numbers** (5-digit, e.g. `66439`) attached to most wholesale
  lines — sequential, almost certainly sourced from an external accounting
  system rather than assigned by hand.
- **Retail chain accounts with formal PO numbers** — multiple Whole Foods
  Market locations appear with `PO: 166823686`-style numbers.
- **A "shares" program** — CSA-style weekly subscriptions, e.g. "50 SHARES...
  19 SHARES: BETloin-off + LSHon", broken out by pickup location (NCFM,
  Durham, HQ, WNC).
- **Out-of-state shipping** — FedEx/UPS references, deliveries to Key West, FL.
- **A satellite Asheville/WNC operation** with its own named van route and a
  distinct set of accounts, run partly independently of the main routes.
- **Standing orders** — recurring items noted separately from the weekly
  order flow, e.g. "STANDING ORDERS: THU - LONGLEAF: 12 CAT".
- **Lot/harvest references inline on order lines** — finfish use the same
  `{SOURCE}-{SPECIES}-{MMDDYY}` pattern seen in the workbook, e.g.
  `(ON-BET-073026)`. Shellfish use a different pattern: a harvest-area
  letter+digit code plus a date, e.g. `H5 8/1`, `D3 8/2`, `G6 8/2` — this
  matches the shape of NC's molluscan shellfish harvest tagging requirement
  and should be treated as regulatory data, not internal shorthand, until
  confirmed otherwise.
- **Nightly EOD (end of day) physical counts**, typed as a flat species+
  quantity list at the close of each day, seen on multiple weekdays in the
  sample — this is the physical-count ritual the rollout plan's phase 3
  cutover is designed to attach to.

## Manual entry vs. formula-computed cells (2026-08-24)

Checked directly by counting `<f>` (formula) elements in the workbook XML,
rather than inferring from appearance.

| Tab | Formula cells | Share |
|---|---|---|
| `Inventory` | 5,343 / 27,775 | ~19% |
| `FF LOG` | 2,037 / 7,201 | ~28% |
| `Frozen Fillets LS - 2026 new` | 144 / 1,241 | ~12% |
| `BULK FRZN SMOKE` | 118 / 1,189 | ~10% |
| `Wanchese Frozen Inv` | 125 / 708 | ~18% |
| `OYP` | 135 / 3,007 | ~4% |
| `OY PLAN` | 8 / 3,194 | <1% |
| `Fish Qty` | 0 / 3,216 | 0% |

**Species codes, lot codes, dock names, treatments, prices, and dates are
hand-typed** — no data validation anywhere, which is what produces the ragged
rows and inconsistent blanks. But real formula infrastructure sits on top:
yield percentages, running sums, cost-per-package, and potential revenue by
channel are computed, not typed. `Inventory` and `FF LOG` carry the heaviest
formula load.

**Tabs are not independent — at least one cross-tab reference exists:** `OYP`
pulls directly from `Inventory` (`Inventory!B475/100`). Any migration or
mirror needs to account for this before assuming a tab can be moved or dropped
in isolation.

**`Fish Qty` is 0% formula despite reading like a pivot summary.** It's either
a pivot table that was "paste special → values"'d at some point, or something
retyped periodically by hand. Either way it's a **frozen snapshot, not a live
view** — a real staleness risk in the current system, and worth naming to the
owner as an example of exactly the kind of bug the new system is meant to
remove.

**Consequence for the importer:** it cannot be "export values, load into
Postgres." It has to distinguish source-of-truth facts (species, lot code,
weight, price) from derived values (yields, potentials) and recompute the
latter rather than copy a formula's cached, possibly-stale output.

## What this means for scope

The original design was framed around "processing team pulls fish for a
restaurant order" — a single channel, single product category, single
location mental model. The real business is a multi-channel wholesale seafood
distributor: wholesale restaurant accounts are the largest single piece, but
retail chain accounts, a subscription program, shipping, and a satellite
location all coexist in the same documents.

This doesn't invalidate the design decisions already made (append-only ledger,
bespoke per-client build, one-way sync, no AI in the core loop, rollout by
narrow slice) — if anything it makes the "narrow first slice" instinct from
the rollout plan more clearly correct. What it does mean: **v1's boundary
needs to be chosen deliberately now**, with the owner, rather than assumed.
See open questions 11–13.
