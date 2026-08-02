# Open questions

Answers go inline under each question, dated. When one is settled and affects
the design, update the spec and note it here as resolved.

## Blocking — schema cannot be finalized without these

### 1. The label printer: make, model, and how is it connected?

Determines the entire print path. A network Zebra speaking ZPL is close to
trivial — the agent opens a socket on port 9100 and writes. A USB Dymo hanging
off someone's laptop is a different and more annoying conversation.

Also needed: what produces labels today, and what does a current label look
like? A photo of one is worth more than a description.

**Answer:**

### 2. Where do orders live today?

"What order it was for" implies a list the crew picks from. Another sheet? An
email inbox? A POS? Verbal?

If orders are not structured anywhere, v1 has to either create an order-entry
screen or let the crew type a restaurant name free-form — a real scope fork.

**Answer:**

### 3. Are lot numbers already tracked in the sheet?

If yes, lots can be seeded from the existing data and receiving capture is less
urgent. If no, an intake screen is unavoidable before labels can carry a lot
number.

What is the lot numbering scheme, and who assigns it?

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

### 7. Units and precision

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
