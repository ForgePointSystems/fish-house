# Open questions

Answers go inline under each question, dated. When one is settled and affects
the design, update the spec and note it here as resolved.

**2026-08-11 — first real documents received.** `data/samples/inventory-2026-08-03.xlsx`
(8-tab inventory workbook) and `data/samples/order-dispatch-doc-2026-08-03.docx`
(weekly order/dispatch log) answered questions 2, 3, and 7 below, and revealed
the business is considerably larger and more multi-channel than the original
framing assumed — see `docs/reference/sheet-findings.md` for the full read and
the spec's "Scope reassessment" section for the consequence. **v1 scope needs a
new conversation before more design work happens.**

## Blocking — schema cannot be finalized without these

### 1. The label printer: make, model, and how is it connected?

Determines the entire print path. A network Zebra speaking ZPL is close to
trivial — the agent opens a socket on port 9100 and writes. A USB Dymo hanging
off someone's laptop is a different and more annoying conversation.

Also needed: what produces labels today, and what does a current label look
like? A photo of one is worth more than a description.

**Answer:** Not in either document. Still needs to be asked directly.

### 2. Where do orders live today?

**Answer (2026-08-11):** Nowhere structured — there is no order database or
form today. Orders live inside a single running Word document per week
(`order-dispatch-doc-2026-08-03.docx`), organized by delivery day, then by
route/driver, then by restaurant, as hand-typed free text: `RESTAURANT NAME
INVOICE# inv: <quantities and species codes>`. The same document also carries
delivery addresses, standing-order notes, driver assignments, and nightly
physical counts. It is simultaneously the order log, the dispatch sheet, and
the count log — not a clean list a UI could pick from.

This changes the shape of "order entry" for v1: there is no existing
structured order to link a pull to. Either the app becomes where orders get
entered in the first place, or v1 pulls are linked to a free-text
customer/invoice reference rather than a real order record. See scope
reassessment.

### 3. Are lot numbers already tracked in the sheet?

**Answer (2026-08-11):** Yes, extensively, in two different places:

- The `Inventory` tab has an explicit **Histamine Lot Code** column, format
  `{SOURCE}-{SPECIES}-{MMDDYY}` (e.g. `PDL-ALM-080326` = Peddler dock, Almaco
  jack, 08/03/26).
- Nearly every line in the order/dispatch doc carries an inline lot or harvest
  reference, e.g. `(ON-BET-073026)` for finfish or `H5 8/1` for shellfish (a
  harvest-area code plus date — this format matches NC's molluscan shellfish
  tagging requirement, i.e. this is already regulatory traceability data, not
  just internal bookkeeping).

Lot numbers are not something v1 needs to invent — they need to be **parsed
and centralized** from formats already in daily use. See
`docs/reference/sheet-findings.md`.

**Still open:** who assigns these codes and at what point (at landing? at
purchase?), and whether the two formats (finfish vs. shellfish) are used
consistently enough to parse automatically or need a human step.

## New, raised by reading the actual documents — blocking, scope-level

### 11. Which sales channel is v1 for?

The business is not "a processing room fulfilling restaurant orders." The
order/dispatch doc shows at minimum: wholesale restaurant accounts (100+,
the bulk of the doc), retail chain accounts with formal POs (Whole Foods
Market, multiple stores), a CSA-style "shares" subscription program, farmers
markets, out-of-state shipping (FedEx/UPS, including to Key West FL), and a
satellite Asheville/WNC operation with its own van route. Each has different
data (a PO number vs. an invoice number vs. a share count vs. nothing at all).

**v1 must pick one channel**, almost certainly wholesale restaurant delivery —
it's the largest and most repetitive section of the doc, and closest to the
original "pull for an order" framing. Everything else stays on the sheet/doc
indefinitely, not just for the trial period.

**Answer:**

### 12. Does the product line include shellfish and live product, or finfish only?

The workbook and doc cover whole/fillet finfish, live oysters, clams, live
crabs, live crawfish, and dry goods, each with distinct units (pounds,
count/dozen, gallons for shucked oysters) and distinct handling. Finfish pulls
and oyster/clam pulls are not the same transaction shape.

**Recommendation, not yet confirmed with the owner:** finfish only for v1. It's
the ledger-and-weight model the design already assumes; shellfish introduces
count-based units and a different traceability tag format that would roughly
double the modeling work for no v1 benefit.

**Answer:**

### 13. Is there a second location (Wanchese) or just a satellite route (Asheville)?

`Wanchese Frozen Inv` is a separate tab in the workbook, suggesting a
second physical inventory location, not just a delivery route. If pulls can
happen from either location, "on hand" is location-scoped, not facility-wide.

**Answer:**

## Important — shape the UX, not the schema

### 4. How many people, and how does data get entered today?

One person at a desk typing from paper tally sheets, or several people entering
directly? Drives whether the tablet needs per-user attribution and how much
concurrent-entry pressure there is.

**Answer:**

### 5. Wifi quality in the processing room and near the freezers

Decides whether the offline queue is a v1 requirement or a genuine deferral.

**Answer:**

### 6. Who owns the Supabase and Vercel accounts?

Recommendation is the business, with us as members. Needs an actual decision
before anything is provisioned, because moving it later is painful.

**Answer:**

### 7. Do they do periodic physical inventory counts, and how often?

**Answer (2026-08-11):** Yes — at minimum a nightly "EOD" (end of day) count,
typed as a flat species-and-quantity list at the bottom of each day's section
in the order/dispatch doc (e.g. "93 mbl 8/6-ON", "245 LSH"). Seen for multiple
weekdays in the sample, so this looks like a daily habit, not occasional. This
is exactly the reconciliation point the rollout plan wants — confirms the
approach, no design change needed. Still worth confirming verbally that EOD
counts happen every day without exception, and whether Saturday/Sunday follow
the same pattern (the doc's retail/farmers-market days look less structured).

### 8. What is the natural first slice?

Phase 2 needs a boundary that is physically obvious to the crew — one species,
one freezer, one shift. Ideally something high enough volume to exercise the app
properly but low enough stakes that two weeks of dual entry is tolerable.

**Answer:**

### 9. Who on the crew would be the first user, and are they willing?

Dual entry is real extra work for a real person. The rollout depends on one
person being bought in rather than resentful. Worth knowing who that is before
promising a timeline.

**Answer:**

### 10. Units and precision

Pounds to one decimal? Whole pounds? Do they ever work in kilos or count
(pieces, cases)? Rounding rules on partial pulls.

**Answer:**

## Needed to write the schema, answered by the sheet itself

- Tab structure and what each tab is for
- How species and product forms are named (whole, fillet, portion, H&G)
- Whether yields are tracked, and how
- Where formulas are load-bearing vs. decorative
- How freezer or cooler locations are recorded, if at all

Get a snapshot export rather than live access — see `README.md`.
