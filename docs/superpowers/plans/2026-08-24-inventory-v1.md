# Inventory v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the facility's hand-maintained `Inventory` tab with a real
append-only ledger, a UI to view on-hand stock and record receiving/
adjustments, an importer that seeds the ledger from the real spreadsheet, and
a one-way mirror of current on-hand back into a Google Sheet.

**Architecture:** Next.js (App Router) on Vercel talking to a Supabase
Postgres project. Every inventory movement is an immutable row in
`inventory_transaction`; on-hand is always a `sum()` view, never a stored
column. A one-off Node script seeds the ledger from a spreadsheet snapshot.
A Supabase Edge Function on a `pg_cron` schedule mirrors current on-hand into
a dedicated tab of the facility's live Google Sheet. No orders, no pulls, no
label printing — see the design spec's "Scope reassessment."

**Tech Stack:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4,
Supabase (Postgres 17, Auth, Edge Functions), Vitest, `xlsx` (SheetJS) for
reading the spreadsheet snapshot, `tsx` for running the import script.

## Global Constraints

Copied verbatim from `CLAUDE.md` and the design spec — every task's work
implicitly includes these:

- **Inventory is an append-only ledger.** Every receipt and adjustment is an
  immutable row in `inventory_transaction`. On-hand is a derived view. Never
  add a mutable `quantity_on_hand` column anywhere.
- **Never block the floor.** An adjustment that would take on-hand negative is
  recorded anyway, not rejected. Negative on-hand is a signal to reconcile,
  not a validation error.
- **No LLM anywhere in this plan's scope.** Every step here is deterministic
  code.
- **Ledger rows carry a `source`** (`app_entered`, `sheet_imported`,
  `count_adjustment`) on every row, from the first migration.
- **No `facility_id`, no tenant discriminator.** This is bespoke software for
  one facility — see `species.category` and `locations` below for how the
  two genuinely-open questions (12, 13) are handled instead.
- **Use the crew's vocabulary** — see `docs/reference/glossary.md`. Where it
  conflicts with a term below, the glossary wins and the code should be
  updated to match once confirmed.
- **Sheet sync is one-way.** The mirror (Task 10) only ever writes to a
  dedicated `FROM_APP` tab. It never reads from the sheet, and nothing in
  this plan writes to the sheet's working tabs.

### How this plan handles open questions 12 and 13

Both are genuinely unanswered by the owner as of this writing
(`docs/reference/open-questions.md`). Rather than block the whole plan on a
conversation that hasn't happened, the schema in Task 2 is built so **either
answer is cheap to accommodate later, and v1 proceeds on the recommended
default in the meantime**:

- **Question 12 (finfish only vs. shellfish too):** `species.category` is a
  plain `text` column, no `CHECK` constraint restricting its values. v1's
  importer and UI only ever write `'finfish'`. If the answer turns out to be
  "yes, shellfish too," adding shellfish species is a data-entry problem, not
  a migration.
- **Question 13 (is Wanchese a second location):** `locations` is a real
  table from the first migration, seeded with exactly one row (`'main'`). If
  Wanchese turns out to be a second physical location, adding it is a second
  seed row, not a migration. v1's importer and UI only ever use `'main'`.

**Recommendation: have this conversation with the owner in parallel with
implementation, not before it.** Nothing in Tasks 1–9 needs the answer.

---

### Task 1: Scaffold the Next.js app and the local Supabase project

**Files:**
- Create: `web/package.json`
- Create: `web/tsconfig.json`
- Create: `web/next.config.ts`
- Create: `web/eslint.config.mjs`
- Create: `web/postcss.config.mjs`
- Create: `web/vitest.config.ts`
- Create: `web/.env.example`
- Create: `web/.gitignore`
- Create: `web/src/app/layout.tsx`
- Create: `web/src/app/globals.css`
- Create: `web/src/app/page.tsx`
- Create: `supabase/config.toml`
- Create: `supabase/seed.sql`
- Modify: `.gitignore` (repo root) — add `web/node_modules`, `web/.next`, `web/.env.local`

**Interfaces:**
- Produces: a working `npm run dev` at `web/` serving `http://localhost:3000`,
  and a working local Supabase stack via `supabase start` from the repo root.
  All later tasks build inside `web/src/`.

- [ ] **Step 1: Create `web/package.json`**

```json
{
  "name": "fish-house-web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "test": "vitest run",
    "import:inventory": "tsx scripts/import-inventory-snapshot.ts"
  },
  "dependencies": {
    "@supabase/ssr": "^0.10.2",
    "@supabase/supabase-js": "^2.103.0",
    "next": "16.2.3",
    "react": "19.2.4",
    "react-dom": "19.2.4"
  },
  "devDependencies": {
    "@eslint/eslintrc": "^3",
    "@tailwindcss/postcss": "^4",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "eslint": "^9",
    "eslint-config-next": "16.2.3",
    "tailwindcss": "^4",
    "typescript": "^5",
    "tsx": "^4",
    "vitest": "^3",
    "xlsx": "^0.18.5"
  }
}
```

- [ ] **Step 2: Create `web/tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts",
    "**/*.mts"
  ],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: Create `web/next.config.ts`**

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;
```

- [ ] **Step 4: Create `web/eslint.config.mjs`**

```js
import { dirname } from "path";
import { fileURLToPath } from "url";
import { FlatCompat } from "@eslint/eslintrc";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const compat = new FlatCompat({
  baseDirectory: __dirname,
});

const eslintConfig = [
  ...compat.extends("next/core-web-vitals", "next/typescript"),
];

export default eslintConfig;
```

- [ ] **Step 5: Create `web/postcss.config.mjs`**

```js
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default config;
```

- [ ] **Step 6: Create `web/vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
  },
});
```

- [ ] **Step 7: Create `web/.env.example`**

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
```

- [ ] **Step 8: Create `web/.gitignore`**

```
node_modules
.next
.env.local
```

- [ ] **Step 9: Create `web/src/app/globals.css`**

```css
@import "tailwindcss";

:root {
  --background: #ffffff;
  --foreground: #171717;
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: #0a0a0a;
    --foreground: #ededed;
  }
}

body {
  background: var(--background);
  color: var(--foreground);
  font-family: Arial, Helvetica, sans-serif;
}
```

- [ ] **Step 10: Create `web/src/app/layout.tsx`**

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "Fish House — Inventory",
  description: "Inventory ledger for the facility",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en" className="h-full antialiased">
      <body className="min-h-full flex flex-col">{children}</body>
    </html>
  );
}
```

- [ ] **Step 11: Create `web/src/app/page.tsx`** (placeholder — Task 7 replaces this)

```tsx
export default function Home() {
  return (
    <main className="flex flex-1 items-center justify-center p-8">
      <p className="text-lg text-zinc-600">Fish House — inventory v1 in progress.</p>
    </main>
  );
}
```

- [ ] **Step 12: Install dependencies and verify the dev server**

Run (from `web/`): `npm install`
Run: `npm run dev`
Expected: server starts on `http://localhost:3000`; visiting it shows "Fish
House — inventory v1 in progress." Stop the server (Ctrl+C) once confirmed.

- [ ] **Step 13: Create `supabase/config.toml`**

```toml
project_id = "fish-house"

[api]
enabled = true
port = 54321
schemas = ["public", "graphql_public"]
extra_search_path = ["public", "extensions"]
max_rows = 1000

[api.tls]
enabled = false

[db]
port = 54322
shadow_port = 54320
health_timeout = "2m"
major_version = 17

[db.pooler]
enabled = false
port = 54329
pool_mode = "transaction"
default_pool_size = 20
max_client_conn = 100

[db.migrations]
enabled = true
schema_paths = []

[db.seed]
enabled = true
sql_paths = ["./seed.sql"]

[db.network_restrictions]
enabled = false
allowed_cidrs = ["0.0.0.0/0"]
allowed_cidrs_v6 = ["::/0"]

[realtime]
enabled = true

[studio]
enabled = true
port = 54323
api_url = "http://127.0.0.1"

[inbucket]
enabled = true
port = 54324

[storage]
enabled = true
file_size_limit = "50MiB"

[storage.s3_protocol]
enabled = true

[storage.analytics]
enabled = false
max_namespaces = 5
max_tables = 10
max_catalogs = 2

[storage.vector]
enabled = false
max_buckets = 10
max_indexes = 5

[auth]
enabled = true
site_url = "http://127.0.0.1:3000"
additional_redirect_urls = ["https://127.0.0.1:3000"]
jwt_expiry = 3600
enable_refresh_token_rotation = true
refresh_token_reuse_interval = 10
enable_signup = true
enable_anonymous_sign_ins = false
enable_manual_linking = false
minimum_password_length = 6
password_requirements = ""

[auth.rate_limit]
email_sent = 2
sms_sent = 30
anonymous_users = 30
token_refresh = 150
sign_in_sign_ups = 30
token_verifications = 30
web3 = 30

[auth.email]
enable_signup = true
double_confirm_changes = true
enable_confirmations = false
secure_password_change = false
max_frequency = "1s"
otp_length = 6
otp_expiry = 3600

[auth.sms]
enable_signup = false
enable_confirmations = false
template = "Your code is {{ .Code }}"
max_frequency = "5s"

[auth.sms.twilio]
enabled = false
account_sid = ""
message_service_sid = ""
auth_token = "env(SUPABASE_AUTH_SMS_TWILIO_AUTH_TOKEN)"

[auth.mfa]
max_enrolled_factors = 10

[auth.mfa.totp]
enroll_enabled = false
verify_enabled = false

[auth.mfa.phone]
enroll_enabled = false
verify_enabled = false
otp_length = 6
template = "Your code is {{ .Code }}"
max_frequency = "5s"

[auth.external.apple]
enabled = false
client_id = ""
secret = "env(SUPABASE_AUTH_EXTERNAL_APPLE_SECRET)"
redirect_uri = ""
url = ""
skip_nonce_check = false
email_optional = false

[auth.web3.solana]
enabled = false

[auth.third_party.firebase]
enabled = false

[auth.third_party.auth0]
enabled = false

[auth.third_party.aws_cognito]
enabled = false

[auth.third_party.clerk]
enabled = false

[auth.oauth_server]
enabled = false
authorization_url_path = "/oauth/consent"
allow_dynamic_registration = false

[edge_runtime]
enabled = true
policy = "per_worker"
inspector_port = 8083
deno_version = 2

[analytics]
enabled = true
port = 54327
backend = "postgres"

[experimental]
orioledb_version = ""
s3_host = "env(S3_HOST)"
s3_region = "env(S3_REGION)"
s3_access_key = "env(S3_ACCESS_KEY)"
s3_secret_key = "env(S3_SECRET_KEY)"
```

- [ ] **Step 14: Create `supabase/seed.sql`** (empty for now — Task 2 fills it in)

```sql
-- Seed data. Task 2 adds the default location here.
```

- [ ] **Step 15: Update root `.gitignore`**

Add these lines if not already present (check first — the repo already
gitignores `data/`, `.env`, `node_modules/`, `.next/`, `supabase/.branches/`,
`supabase/.temp/`, `.vercel`):

```
web/node_modules
web/.next
web/.env.local
```

- [ ] **Step 16: Verify the local Supabase stack starts**

Run (from repo root, requires Docker Desktop running):
`supabase start`
Expected: after downloading images on first run, it prints local URLs
including `API URL`, `anon key`, and `service_role key`. Leave it running —
later tasks need it.

- [ ] **Step 17: Create `web/.env.local`** (not committed — gitignored)

Copy the `API URL` as `NEXT_PUBLIC_SUPABASE_URL`, `anon key` as
`NEXT_PUBLIC_SUPABASE_ANON_KEY`, and `service_role key` as
`SUPABASE_SERVICE_ROLE_KEY` from the `supabase start` output into
`web/.env.local`, following the shape of `web/.env.example`.

- [ ] **Step 18: Commit**

```bash
git add web/package.json web/tsconfig.json web/next.config.ts web/eslint.config.mjs web/postcss.config.mjs web/vitest.config.ts web/.env.example web/.gitignore web/src/app/layout.tsx web/src/app/globals.css web/src/app/page.tsx web/package-lock.json supabase/config.toml supabase/seed.sql .gitignore
git commit -m "Scaffold Next.js app and local Supabase project"
```

---

### Task 2: Core schema — species, locations, lots, the ledger, and on-hand views

**Files:**
- Create: `supabase/migrations/20260824120000_inventory_core.sql`
- Modify: `supabase/seed.sql`
- Create: `web/src/types/database.types.ts` (generated)
- Create: `web/vitest.setup.env.ts`
- Create: `web/src/lib/supabase/service.ts`
- Test: `web/src/lib/inventory/schema.integration.test.ts`

**Interfaces:**
- Consumes: local Supabase stack running (Task 1, Step 16), `web/.env.local`
  populated (Task 1, Step 17).
- Produces: tables `species`, `locations`, `lots`, `inventory_transaction`;
  views `current_on_hand`, `on_hand_by_species`. `createServiceRoleClient()`
  from `web/src/lib/supabase/service.ts`, used by every later task that needs
  to bypass RLS (the import script in Task 6).

- [ ] **Step 1: Create the migration**

`supabase/migrations/20260824120000_inventory_core.sql`:

```sql
-- Core inventory ledger: species, locations, lots, and the append-only
-- transaction log. On-hand is always derived from the ledger via the views
-- at the end of this file -- never a stored, mutable column. See
-- CLAUDE.md's "Inventory is an append-only ledger" rule.

create table species (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  name text not null,
  -- No CHECK on category: v1 only ever writes 'finfish', but shellfish is
  -- a real, evidenced future need (open question 12), not a hypothetical
  -- one. Leaving this open costs nothing now and avoids a migration later.
  category text not null default 'finfish',
  created_at timestamptz not null default now()
);

create table locations (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  name text not null,
  created_at timestamptz not null default now()
);

create table lots (
  id uuid primary key default gen_random_uuid(),
  species_id uuid not null references species(id),
  location_id uuid not null references locations(id),
  -- Verbatim from the source, e.g. PDL-ALM-080326. Not parsed into columns
  -- -- see docs/reference/glossary.md. Nullable: not every row in the
  -- source sheet has one.
  lot_code text,
  landed_at text,
  landed_date date,
  created_at timestamptz not null default now()
);

create index lots_species_id_idx on lots (species_id);
create index lots_location_id_idx on lots (location_id);

create table inventory_transaction (
  id uuid primary key default gen_random_uuid(),
  lot_id uuid not null references lots(id),
  transaction_type text not null check (transaction_type in ('receipt', 'adjustment')),
  -- Signed: positive adds to on-hand, negative removes. A single column
  -- keeps on-hand a plain sum().
  quantity numeric not null,
  -- Constrained on purpose, unlike species.category: v1's UI and importer
  -- only ever produce 'lb'. Loosening this is one migration line when a
  -- second unit is actually needed.
  unit text not null default 'lb' check (unit in ('lb')),
  reason text check (reason is null or reason in ('sold', 'used', 'spoiled', 'correction')),
  -- Distinguishes what the app recorded from what was imported from the
  -- sheet -- required by CLAUDE.md, load-bearing for the future mirror
  -- and divergence report.
  source text not null check (source in ('app_entered', 'sheet_imported', 'count_adjustment')),
  note text,
  occurred_at timestamptz not null default now(),
  created_at timestamptz not null default now(),
  created_by uuid references auth.users(id)
);

create index inventory_transaction_lot_id_idx on inventory_transaction (lot_id);
create index inventory_transaction_occurred_at_idx on inventory_transaction (occurred_at desc);

-- Per-lot on-hand. Never stored -- always summed from the ledger.
create view current_on_hand as
select
  l.id as lot_id,
  l.species_id,
  s.code as species_code,
  s.name as species_name,
  l.location_id,
  loc.code as location_code,
  l.lot_code,
  l.landed_at,
  l.landed_date,
  coalesce(sum(t.quantity), 0) as on_hand
from lots l
join species s on s.id = l.species_id
join locations loc on loc.id = l.location_id
left join inventory_transaction t on t.lot_id = l.id
group by l.id, l.species_id, s.code, s.name, l.location_id, loc.code, l.lot_code, l.landed_at, l.landed_date;

-- Species-level rollup. This replaces the stale `Fish Qty` pivot -- unlike
-- that tab, this is always live because it is a view, not a snapshot.
create view on_hand_by_species as
select
  species_id,
  species_code,
  species_name,
  sum(on_hand) as total_on_hand
from current_on_hand
group by species_id, species_code, species_name;
```

- [ ] **Step 2: Seed the default location**

Update `supabase/seed.sql`:

```sql
-- Seed data.

-- v1 assumes a single location. See open question 13 -- if the owner
-- confirms Wanchese is a second real location, add a second row here;
-- no migration required.
insert into locations (code, name) values ('main', 'Main facility');
```

- [ ] **Step 3: Apply the migration locally and verify**

Run (from repo root): `supabase db reset`
Expected: output shows migrations applying, then seed running, ending
"Finished supabase db reset". No errors.

Run: `supabase db diff --local`
Expected: no output (schema matches the applied migration — confirms the
migration file is the source of truth).

- [ ] **Step 4: Generate TypeScript types**

Run (from `web/`): `supabase gen types typescript --local > src/types/database.types.ts`
Expected: the file is created and non-empty, containing a `Database` type
with `species`, `locations`, `lots`, `inventory_transaction` tables and
`current_on_hand`, `on_hand_by_species` views under `public`.

- [ ] **Step 5: Create the service-role Supabase client**

`web/src/lib/supabase/service.ts`:

```ts
import { createClient } from "@supabase/supabase-js";
import type { Database } from "@/types/database.types";

// Bypasses RLS. Only for trusted server-side contexts that are not
// answering a specific user's request: the import script (Task 6) and any
// future scheduled job. Never import this into a Server Action or a
// component that runs on behalf of a logged-in user -- use
// web/src/lib/supabase/server.ts for that.
export function createServiceRoleClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );
}
```

- [ ] **Step 6: Create a test env loader**

`web/vitest.setup.env.ts`:

```ts
import { config } from "dotenv";
import { resolve } from "path";

config({ path: resolve(__dirname, ".env.local") });
```

Update `web/vitest.config.ts` to load it:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    setupFiles: ["./vitest.setup.env.ts"],
  },
});
```

Run (from `web/`): `npm install dotenv --save-dev`

- [ ] **Step 7: Write the failing integration test**

`web/src/lib/inventory/schema.integration.test.ts`:

```ts
import { describe, it, expect, beforeAll } from "vitest";
import { createServiceRoleClient } from "@/lib/supabase/service";

describe("inventory core schema", () => {
  const supabase = createServiceRoleClient();
  let speciesId: string;
  let locationId: string;
  let lotId: string;

  beforeAll(async () => {
    const { data: location, error: locationError } = await supabase
      .from("locations")
      .select("id")
      .eq("code", "main")
      .single();
    if (locationError) throw locationError;
    locationId = location.id;

    const { data: species, error: speciesError } = await supabase
      .from("species")
      .insert({ code: "TESTFISH", name: "Test Fish" })
      .select("id")
      .single();
    if (speciesError) throw speciesError;
    speciesId = species.id;

    const { data: lot, error: lotError } = await supabase
      .from("lots")
      .insert({
        species_id: speciesId,
        location_id: locationId,
        lot_code: "TST-TESTFISH-010126",
        landed_at: "Test Dock",
        landed_date: "2026-01-01",
      })
      .select("id")
      .single();
    if (lotError) throw lotError;
    lotId = lot.id;
  });

  it("sums receipts and adjustments into current_on_hand", async () => {
    await supabase.from("inventory_transaction").insert([
      {
        lot_id: lotId,
        transaction_type: "receipt",
        quantity: 100,
        source: "sheet_imported",
      },
      {
        lot_id: lotId,
        transaction_type: "adjustment",
        quantity: -30,
        reason: "sold",
        source: "app_entered",
      },
    ]);

    const { data, error } = await supabase
      .from("current_on_hand")
      .select("on_hand")
      .eq("lot_id", lotId)
      .single();

    expect(error).toBeNull();
    expect(data?.on_hand).toBe(70);
  });

  it("rolls lots up into on_hand_by_species", async () => {
    const { data, error } = await supabase
      .from("on_hand_by_species")
      .select("total_on_hand")
      .eq("species_id", speciesId)
      .single();

    expect(error).toBeNull();
    expect(data?.total_on_hand).toBe(70);
  });

  it("allows an adjustment to take on-hand negative without rejecting it", async () => {
    await supabase.from("inventory_transaction").insert({
      lot_id: lotId,
      transaction_type: "adjustment",
      quantity: -1000,
      reason: "correction",
      source: "app_entered",
    });

    const { data, error } = await supabase
      .from("current_on_hand")
      .select("on_hand")
      .eq("lot_id", lotId)
      .single();

    expect(error).toBeNull();
    expect(Number(data?.on_hand)).toBeLessThan(0);
  });
});
```

- [ ] **Step 8: Run the test to verify it fails**

Run (from `web/`): `npx vitest run src/lib/inventory/schema.integration.test.ts`
Expected: FAIL — `Cannot find module '@/lib/supabase/service'` or similar,
since Step 5's file doesn't exist until you've done it, or a connection
error if `web/.env.local` isn't populated yet. If it's a missing-module
error, confirm Step 5 was completed; if it's a connection error, confirm
`supabase start` (Task 1, Step 16) is still running and `.env.local` values
are correct.

- [ ] **Step 9: Run the test to verify it passes**

Run: `npx vitest run src/lib/inventory/schema.integration.test.ts`
Expected: PASS, 3 tests.

- [ ] **Step 10: Commit**

```bash
git add supabase/migrations/20260824120000_inventory_core.sql supabase/seed.sql web/src/types/database.types.ts web/src/lib/supabase/service.ts web/vitest.setup.env.ts web/vitest.config.ts web/src/lib/inventory/schema.integration.test.ts web/package.json web/package-lock.json
git commit -m "Add core inventory schema: species, locations, lots, ledger, on-hand views"
```

---

### Task 3: Auth — profiles and RLS

**Files:**
- Create: `supabase/migrations/20260824120100_profiles_and_rls.sql`
- Test: `web/src/lib/inventory/rls.integration.test.ts`

**Interfaces:**
- Consumes: schema from Task 2.
- Produces: `profiles` table, `is_provisioned_user()` function, RLS policies
  on `species`, `locations`, `lots`, `inventory_transaction`, `profiles`.
  Later tasks' Server Actions rely on RLS allowing any provisioned
  (authenticated + has a `profiles` row) user to read and write.

- [ ] **Step 1: Create the migration**

`supabase/migrations/20260824120100_profiles_and_rls.sql`:

```sql
-- profiles: one row per real user. Created manually via SQL for now (see
-- README below) -- v1 has few enough users that a signup trigger isn't
-- worth the complexity yet.
--
-- Its only job right now is to gate access: policies below check "does a
-- profile exist for this user," not the role, because no v1 feature is
-- owner-only yet. The role column exists so tightening a specific policy
-- later is a one-line change, not a schema migration.
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  role text not null default 'crew' check (role in ('owner', 'crew')),
  created_at timestamptz not null default now()
);

alter table species enable row level security;
alter table locations enable row level security;
alter table lots enable row level security;
alter table inventory_transaction enable row level security;
alter table profiles enable row level security;

create function is_provisioned_user()
returns boolean
language sql
security definer
stable
as $$
  select exists (select 1 from profiles where id = auth.uid());
$$;

create policy "provisioned users can read species"
  on species for select
  using (is_provisioned_user());

create policy "provisioned users can insert species"
  on species for insert
  with check (is_provisioned_user());

create policy "provisioned users can read locations"
  on locations for select
  using (is_provisioned_user());

create policy "provisioned users can read lots"
  on lots for select
  using (is_provisioned_user());

create policy "provisioned users can insert lots"
  on lots for insert
  with check (is_provisioned_user());

create policy "provisioned users can read transactions"
  on inventory_transaction for select
  using (is_provisioned_user());

create policy "provisioned users can insert transactions"
  on inventory_transaction for insert
  with check (is_provisioned_user());

create policy "users can read their own profile"
  on profiles for select
  using (id = auth.uid());
```

- [ ] **Step 2: Apply and verify**

Run: `supabase db reset`
Expected: applies cleanly, no errors.

- [ ] **Step 3: Regenerate types**

Run (from `web/`): `supabase gen types typescript --local > src/types/database.types.ts`
Expected: `profiles` now appears in the generated `Database` type.

- [ ] **Step 4: Write the failing test**

`web/src/lib/inventory/rls.integration.test.ts`:

```ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { createClient } from "@supabase/supabase-js";
import { createServiceRoleClient } from "@/lib/supabase/service";
import type { Database } from "@/types/database.types";

describe("RLS on inventory tables", () => {
  const admin = createServiceRoleClient();
  const anon = createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );

  let provisionedUserClient: ReturnType<typeof createClient<Database>>;
  const testEmail = `rls-test-${Date.now()}@example.com`;
  const testPassword = "test-password-123";
  let userId: string;

  beforeAll(async () => {
    const { data, error } = await admin.auth.admin.createUser({
      email: testEmail,
      password: testPassword,
      email_confirm: true,
    });
    if (error) throw error;
    userId = data.user.id;

    const { error: profileError } = await admin
      .from("profiles")
      .insert({ id: userId, role: "crew" });
    if (profileError) throw profileError;

    provisionedUserClient = createClient<Database>(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
    );
    const { error: signInError } = await provisionedUserClient.auth.signInWithPassword({
      email: testEmail,
      password: testPassword,
    });
    if (signInError) throw signInError;
  });

  afterAll(async () => {
    await admin.auth.admin.deleteUser(userId);
  });

  it("blocks anonymous reads of locations", async () => {
    const { data, error } = await anon.from("locations").select("*");
    expect(error).toBeNull();
    expect(data).toEqual([]);
  });

  it("allows a provisioned user to read locations", async () => {
    const { data, error } = await provisionedUserClient.from("locations").select("*");
    expect(error).toBeNull();
    expect(data!.length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 5: Run the test to verify it fails**

Run (from `web/`): `npx vitest run src/lib/inventory/rls.integration.test.ts`
Expected: FAIL on the "blocks anonymous reads" case if RLS wasn't applied
correctly, or a connection/setup error if something above wasn't completed.
Confirm it fails for the right reason before proceeding.

- [ ] **Step 6: Run the test to verify it passes**

Run: `npx vitest run src/lib/inventory/rls.integration.test.ts`
Expected: PASS, 2 tests.

- [ ] **Step 7: Commit**

```bash
git add supabase/migrations/20260824120100_profiles_and_rls.sql web/src/types/database.types.ts web/src/lib/inventory/rls.integration.test.ts
git commit -m "Add profiles table and RLS policies gating access to provisioned users"
```

---

### Task 4: Excel serial date conversion

**Files:**
- Create: `web/src/lib/import/excelDate.ts`
- Test: `web/src/lib/import/excelDate.test.ts`

**Interfaces:**
- Produces: `excelSerialToDate(serial: number): Date` — used by Task 5's
  parser to convert the sheet's `Pickup Date` column.

- [ ] **Step 1: Write the failing test**

`web/src/lib/import/excelDate.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { excelSerialToDate } from "./excelDate";

describe("excelSerialToDate", () => {
  it("converts serial 25569 to the Unix epoch (definitional: the standard offset)", () => {
    const result = excelSerialToDate(25569);
    expect(result.toISOString()).toBe("1970-01-01T00:00:00.000Z");
  });

  it("converts a real serial from the sheet (46237) to a date in August 2026", () => {
    const result = excelSerialToDate(46237);
    expect(result.getUTCFullYear()).toBe(2026);
    expect(result.getUTCMonth()).toBe(7); // 0-indexed: 7 = August
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run (from `web/`): `npx vitest run src/lib/import/excelDate.test.ts`
Expected: FAIL with "Cannot find module './excelDate'".

- [ ] **Step 3: Write the implementation**

`web/src/lib/import/excelDate.ts`:

```ts
// Excel (and Google Sheets' xlsx export) stores dates as a serial number of
// days since 1899-12-30 -- the "1900 date system," including its well-known
// leap-year bug. 25569 is the standard, well-documented offset to the Unix
// epoch: (serial - 25569) days since 1970-01-01.
export function excelSerialToDate(serial: number): Date {
  const msPerDay = 86400 * 1000;
  return new Date((serial - 25569) * msPerDay);
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/lib/import/excelDate.test.ts`
Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add web/src/lib/import/excelDate.ts web/src/lib/import/excelDate.test.ts
git commit -m "Add Excel serial date conversion"
```

---

### Task 5: Parse the `Inventory` tab into structured lot rows

**Files:**
- Create: `web/src/lib/import/parseInventorySheet.ts`
- Test: `web/src/lib/import/parseInventorySheet.test.ts`

**Interfaces:**
- Consumes: `excelSerialToDate` from Task 4.
- Produces: `parseInventorySheet(rows: unknown[][]): ParsedLotRow[]` and the
  `ParsedLotRow` type — consumed by Task 6's import script.

**Column mapping**, confirmed against the real snapshot in
`docs/reference/sheet-findings.md` (row 0 is a dashboard row, row 1 is a
category-label row, row 2 is the real header, data starts at row 3):

| Index | Column | Field |
|---|---|---|
| 0 | `lbs remain` | weight |
| 1 | `Code` | species code |
| 3 | `Species` | species name |
| 6 | `Histamine Lot Code` | lot code |
| 7 | `Landed at` | landed at |
| 9 | `Pickup Date` | landed date (Excel serial) |

Yield percentages, per-cut pricing, and revenue-potential columns (indices
10–21 and beyond) are formula-derived and are **not imported in v1** — no v1
screen displays them (see `docs/reference/sheet-findings.md`'s formula
analysis). This isn't data loss: the original spreadsheet remains available,
and these can be added as computed views over the ledger later, once a
screen needs them.

A row is skipped if its species code (index 1) is blank, or if both weight
(index 0) is zero and lot code (index 6) is blank — these are leftover
template rows in the sheet, not real lots.

- [ ] **Step 1: Write the failing test**

`web/src/lib/import/parseInventorySheet.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { parseInventorySheet } from "./parseInventorySheet";

describe("parseInventorySheet", () => {
  const dashboardRow = [" ", "2323.040565", "", "WHOLE FISH REMAINING"];
  const categoryRow = ["INVENTORY", "", "", "", "", "", "", "", "", "", "Yields"];
  const headerRow = [
    "lbs remain", "Code", "", "Species", "", "", "Histamine Lot Code",
    "Landed at", "", "Pickup Date", "s-on %", "off %", "s/g %", "s-on Y ",
    "s/l Y", "s/g Y", "s-on $", "0.4", "s/l $", "0.4", "s/g $", "0.35",
    "Pounds", "", "", "Size Range", "On", "Off",
  ];
  const realLotRow = [
    "0.1162790698", "ALMACO", "", "Almaco Jack", "", "", "PDL-ALM-080326",
    "Peddler (Holden Beach, NC)", "", "46237.0", "0.43", "0.37", "1.0",
    "0.05", "0.04302325581", "0.1162790698", "11.81395349", "9.689922481",
    "12.75675676", "16.89189189", "5.25", "3.846153846", "135", "2.5", "",
    "44691.0", "58.0",
  ];
  const blankTemplateRow = [
    "0", "ALMACO", "", "Almaco Jack", "", "", "", "", "", "", "0.43",
    "0.37", "1.0", "0", "0", "0", "6", "0", "6", "0", "2.75", "0",
  ];
  const blankCodeRow = ["", "", "", "", "", "", "", "", "", ""];

  const rows = [dashboardRow, categoryRow, headerRow, realLotRow, blankTemplateRow, blankCodeRow];

  it("extracts one lot from a real data row", () => {
    const result = parseInventorySheet(rows);
    expect(result).toHaveLength(1);
    expect(result[0]).toEqual({
      speciesCode: "ALMACO",
      speciesName: "Almaco Jack",
      lotCode: "PDL-ALM-080326",
      landedAt: "Peddler (Holden Beach, NC)",
      landedDate: new Date((46237 - 25569) * 86400 * 1000),
      weightLbs: 0.1162790698,
    });
  });

  it("skips rows with zero weight and no lot code", () => {
    const result = parseInventorySheet([headerRow, blankTemplateRow]);
    expect(result).toHaveLength(0);
  });

  it("skips rows with a blank species code", () => {
    const result = parseInventorySheet([headerRow, blankCodeRow]);
    expect(result).toHaveLength(0);
  });

  it("treats a missing pickup date as null, not an error", () => {
    const rowWithNoDate = [...realLotRow];
    rowWithNoDate[9] = "";
    const result = parseInventorySheet([headerRow, rowWithNoDate]);
    expect(result[0].landedDate).toBeNull();
  });

  it("treats a missing lot code as null, not an error", () => {
    const rowWithNoLotCode = [...realLotRow];
    rowWithNoLotCode[6] = "";
    const result = parseInventorySheet([headerRow, rowWithNoLotCode]);
    expect(result[0].lotCode).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run (from `web/`): `npx vitest run src/lib/import/parseInventorySheet.test.ts`
Expected: FAIL with "Cannot find module './parseInventorySheet'".

- [ ] **Step 3: Write the implementation**

`web/src/lib/import/parseInventorySheet.ts`:

```ts
import { excelSerialToDate } from "./excelDate";

export interface ParsedLotRow {
  speciesCode: string;
  speciesName: string;
  lotCode: string | null;
  landedAt: string | null;
  landedDate: Date | null;
  weightLbs: number;
}

const COL = {
  WEIGHT: 0,
  CODE: 1,
  SPECIES_NAME: 3,
  LOT_CODE: 6,
  LANDED_AT: 7,
  PICKUP_DATE: 9,
} as const;

const DATA_START_ROW = 3;

function cellText(row: unknown[], index: number): string {
  const value = row[index];
  if (value === undefined || value === null) return "";
  return String(value).trim();
}

function cellNumber(row: unknown[], index: number): number {
  const text = cellText(row, index);
  if (text === "") return 0;
  const parsed = Number(text);
  return Number.isNaN(parsed) ? 0 : parsed;
}

// Parses the raw `Inventory` tab (as an array of rows, each an array of raw
// cell values -- e.g. from `xlsx`'s `sheet_to_json(sheet, { header: 1 })`).
// Extracts only source-of-truth fact columns; formula-derived columns
// (yield %, pricing, revenue potential) are intentionally ignored -- see
// this task's header comment in the plan for why.
export function parseInventorySheet(rows: unknown[][]): ParsedLotRow[] {
  const results: ParsedLotRow[] = [];

  for (let i = DATA_START_ROW; i < rows.length; i++) {
    const row = rows[i];
    if (!row) continue;

    const speciesCode = cellText(row, COL.CODE);
    if (speciesCode === "") continue;

    const weightLbs = cellNumber(row, COL.WEIGHT);
    const lotCodeText = cellText(row, COL.LOT_CODE);
    const lotCode = lotCodeText === "" ? null : lotCodeText;

    if (weightLbs === 0 && lotCode === null) continue;

    const speciesName = cellText(row, COL.SPECIES_NAME);
    const landedAtText = cellText(row, COL.LANDED_AT);
    const landedAt = landedAtText === "" ? null : landedAtText;

    const pickupDateText = cellText(row, COL.PICKUP_DATE);
    const landedDate =
      pickupDateText === "" ? null : excelSerialToDate(Number(pickupDateText));

    results.push({
      speciesCode,
      speciesName,
      lotCode,
      landedAt,
      landedDate,
      weightLbs,
    });
  }

  return results;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/lib/import/parseInventorySheet.test.ts`
Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

```bash
git add web/src/lib/import/parseInventorySheet.ts web/src/lib/import/parseInventorySheet.test.ts
git commit -m "Add Inventory tab parser, extracting facts and skipping derived columns"
```

---

### Task 6: Import script — seed the ledger from a real snapshot

**Files:**
- Create: `web/scripts/import-inventory-snapshot.ts`
- Test: `web/scripts/import-inventory-snapshot.integration.test.ts`

**Interfaces:**
- Consumes: `parseInventorySheet` (Task 5), `createServiceRoleClient` (Task
  2).
- Produces: a runnable CLI (`npm run import:inventory -- <path-to-xlsx>`)
  and an exported `importInventorySnapshot(rows, supabase)` function the
  test calls directly.

**Import semantics** (worth stating precisely, since this is a one-time
seed, not a full historical reconstruction): the snapshot's `lbs remain` for
each lot becomes that lot's starting balance, recorded as a single `receipt`
transaction with `source: 'sheet_imported'`. This is a starting balance, not
a reconstruction of the lot's history before the app existed — consistent
with why `source` exists at all.

- [ ] **Step 1: Write the failing test**

`web/scripts/import-inventory-snapshot.integration.test.ts`:

```ts
import { describe, it, expect, afterEach } from "vitest";
import { createServiceRoleClient } from "@/lib/supabase/service";
import { importInventorySnapshot } from "./import-inventory-snapshot";
import type { ParsedLotRow } from "@/lib/import/parseInventorySheet";

describe("importInventorySnapshot", () => {
  const supabase = createServiceRoleClient();
  const testCode = `IMPTEST${Date.now()}`;

  afterEach(async () => {
    const { data: species } = await supabase
      .from("species")
      .select("id")
      .eq("code", testCode)
      .maybeSingle();
    if (species) {
      const { data: lots } = await supabase.from("lots").select("id").eq("species_id", species.id);
      for (const lot of lots ?? []) {
        await supabase.from("inventory_transaction").delete().eq("lot_id", lot.id);
      }
      await supabase.from("lots").delete().eq("species_id", species.id);
      await supabase.from("species").delete().eq("id", species.id);
    }
  });

  it("creates a species, a lot, and a sheet_imported receipt transaction", async () => {
    const parsedRows: ParsedLotRow[] = [
      {
        speciesCode: testCode,
        speciesName: "Import Test Fish",
        lotCode: "TST-IMPTEST-010126",
        landedAt: "Test Dock",
        landedDate: new Date("2026-01-01T00:00:00.000Z"),
        weightLbs: 42.5,
      },
    ];

    await importInventorySnapshot(parsedRows, supabase);

    const { data: species, error: speciesError } = await supabase
      .from("species")
      .select("id, name")
      .eq("code", testCode)
      .single();
    expect(speciesError).toBeNull();
    expect(species?.name).toBe("Import Test Fish");

    const { data: onHand, error: onHandError } = await supabase
      .from("on_hand_by_species")
      .select("total_on_hand")
      .eq("species_id", species!.id)
      .single();
    expect(onHandError).toBeNull();
    expect(Number(onHand?.total_on_hand)).toBe(42.5);

    const { data: lots } = await supabase
      .from("lots")
      .select("lot_code, landed_at")
      .eq("species_id", species!.id);
    expect(lots).toHaveLength(1);
    expect(lots![0].lot_code).toBe("TST-IMPTEST-010126");

    const { data: transactions } = await supabase
      .from("inventory_transaction")
      .select("source, transaction_type, quantity")
      .eq("lot_id", (await supabase.from("lots").select("id").eq("species_id", species!.id).single()).data!.id);
    expect(transactions).toHaveLength(1);
    expect(transactions![0].source).toBe("sheet_imported");
    expect(transactions![0].transaction_type).toBe("receipt");
    expect(Number(transactions![0].quantity)).toBe(42.5);
  });

  it("reuses an existing species by code rather than duplicating it", async () => {
    const firstBatch: ParsedLotRow[] = [
      {
        speciesCode: testCode,
        speciesName: "Import Test Fish",
        lotCode: "TST-IMPTEST-010126",
        landedAt: "Test Dock",
        landedDate: new Date("2026-01-01T00:00:00.000Z"),
        weightLbs: 10,
      },
    ];
    const secondBatch: ParsedLotRow[] = [
      {
        speciesCode: testCode,
        speciesName: "Import Test Fish",
        lotCode: "TST-IMPTEST-020126",
        landedAt: "Test Dock",
        landedDate: new Date("2026-01-02T00:00:00.000Z"),
        weightLbs: 5,
      },
    ];

    await importInventorySnapshot(firstBatch, supabase);
    await importInventorySnapshot(secondBatch, supabase);

    const { data: speciesRows } = await supabase.from("species").select("id").eq("code", testCode);
    expect(speciesRows).toHaveLength(1);

    const { data: lots } = await supabase.from("lots").select("id").eq("species_id", speciesRows![0].id);
    expect(lots).toHaveLength(2);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run (from `web/`): `npx vitest run scripts/import-inventory-snapshot.integration.test.ts`
Expected: FAIL with "Cannot find module './import-inventory-snapshot'".

- [ ] **Step 3: Write the implementation**

`web/scripts/import-inventory-snapshot.ts`:

```ts
import { readFileSync } from "fs";
import { read, utils } from "xlsx";
import { createServiceRoleClient } from "../src/lib/supabase/service";
import { parseInventorySheet, type ParsedLotRow } from "../src/lib/import/parseInventorySheet";

type SupabaseClient = ReturnType<typeof createServiceRoleClient>;

export async function importInventorySnapshot(
  parsedRows: ParsedLotRow[],
  supabase: SupabaseClient
): Promise<{ lotsCreated: number }> {
  const { data: mainLocation, error: locationError } = await supabase
    .from("locations")
    .select("id")
    .eq("code", "main")
    .single();
  if (locationError) throw locationError;

  const speciesCache = new Map<string, string>();
  let lotsCreated = 0;

  for (const row of parsedRows) {
    let speciesId = speciesCache.get(row.speciesCode);
    if (!speciesId) {
      const { data: existing } = await supabase
        .from("species")
        .select("id")
        .eq("code", row.speciesCode)
        .maybeSingle();

      if (existing) {
        speciesId = existing.id;
      } else {
        const { data: created, error: createError } = await supabase
          .from("species")
          .insert({ code: row.speciesCode, name: row.speciesName })
          .select("id")
          .single();
        if (createError) throw createError;
        speciesId = created.id;
      }
      speciesCache.set(row.speciesCode, speciesId);
    }

    const { data: lot, error: lotError } = await supabase
      .from("lots")
      .insert({
        species_id: speciesId,
        location_id: mainLocation.id,
        lot_code: row.lotCode,
        landed_at: row.landedAt,
        landed_date: row.landedDate ? row.landedDate.toISOString().slice(0, 10) : null,
      })
      .select("id")
      .single();
    if (lotError) throw lotError;

    const { error: transactionError } = await supabase.from("inventory_transaction").insert({
      lot_id: lot.id,
      transaction_type: "receipt",
      quantity: row.weightLbs,
      source: "sheet_imported",
    });
    if (transactionError) throw transactionError;

    lotsCreated++;
  }

  return { lotsCreated };
}

async function main() {
  const filePath = process.argv[2];
  if (!filePath) {
    console.error("Usage: npm run import:inventory -- <path-to-xlsx>");
    process.exit(1);
  }

  const buffer = readFileSync(filePath);
  const workbook = read(buffer, { type: "buffer" });
  const sheet = workbook.Sheets["Inventory"];
  if (!sheet) {
    console.error(`No "Inventory" tab found in ${filePath}`);
    process.exit(1);
  }

  const rows = utils.sheet_to_json<unknown[]>(sheet, { header: 1 });
  const parsedRows = parseInventorySheet(rows);
  console.log(`Parsed ${parsedRows.length} lots from ${filePath}`);

  const supabase = createServiceRoleClient();
  const result = await importInventorySnapshot(parsedRows, supabase);
  console.log(`Created ${result.lotsCreated} lots.`);
}

if (require.main === module) {
  main().catch((error) => {
    console.error(error);
    process.exit(1);
  });
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run scripts/import-inventory-snapshot.integration.test.ts`
Expected: PASS, 2 tests.

- [ ] **Step 5: Run the importer against the real snapshot**

Run (from `web/`): `npm run import:inventory -- ../data/samples/inventory-2026-08-03.xlsx`
Expected: prints a parsed-lot count in the hundreds (the real `Inventory` tab
has ~687 rows, most of which are real lots) and a matching created-lot count.
This is real business data landing in your local Supabase instance — do not
run this against a remote/production project without deliberately deciding
to.

- [ ] **Step 6: Commit**

```bash
git add web/scripts/import-inventory-snapshot.ts web/scripts/import-inventory-snapshot.integration.test.ts
git commit -m "Add import script seeding the ledger from a real Inventory tab snapshot"
```

---

### Task 7: On-hand screen

**Files:**
- Create: `web/src/lib/inventory/queries.ts`
- Modify: `web/src/app/page.tsx`
- Create: `web/src/components/OnHandTable.tsx`
- Create: `web/src/app/api/health/route.ts` (trivial — used only for manual smoke-testing auth in this task)

**Interfaces:**
- Consumes: `on_hand_by_species` view (Task 2), server Supabase client
  (already exists at `web/src/lib/supabase/server.ts` — not modified here).
- Produces: `getOnHandBySpecies(): Promise<OnHandBySpeciesRow[]>`, used by
  Task 9 for the adjustment screen's species picker.

This task's core logic (`getOnHandBySpecies`) is a thin query wrapper with no
independently interesting branching logic to unit test — it's covered by
manual verification against the local stack, consistent with the tiered
testing approach in `docs/superpowers/specs/2026-08-01-fish-house-design.md`'s
sibling project pattern (unit tests for pure logic, manual smoke-test for
thin DB-touching code).

- [ ] **Step 1: Create the query module**

`web/src/lib/inventory/queries.ts`:

```ts
import { createClient } from "@/lib/supabase/server";

export interface OnHandBySpeciesRow {
  speciesId: string;
  speciesCode: string;
  speciesName: string;
  totalOnHand: number;
}

export async function getOnHandBySpecies(): Promise<OnHandBySpeciesRow[]> {
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("on_hand_by_species")
    .select("species_id, species_code, species_name, total_on_hand")
    .order("species_name");

  if (error) throw error;

  return (data ?? []).map((row) => ({
    speciesId: row.species_id,
    speciesCode: row.species_code,
    speciesName: row.species_name,
    totalOnHand: Number(row.total_on_hand),
  }));
}
```

- [ ] **Step 2: Create the on-hand table component**

`web/src/components/OnHandTable.tsx`:

```tsx
import type { OnHandBySpeciesRow } from "@/lib/inventory/queries";

export function OnHandTable({ rows }: { rows: OnHandBySpeciesRow[] }) {
  if (rows.length === 0) {
    return <p className="text-zinc-600">No inventory on hand.</p>;
  }

  return (
    <table className="w-full text-left border-collapse">
      <thead>
        <tr className="border-b border-zinc-300">
          <th className="py-2 pr-4">Species</th>
          <th className="py-2 pr-4">Code</th>
          <th className="py-2 text-right">On hand (lb)</th>
        </tr>
      </thead>
      <tbody>
        {rows.map((row) => (
          <tr key={row.speciesId} className="border-b border-zinc-100">
            <td className="py-2 pr-4">{row.speciesName}</td>
            <td className="py-2 pr-4 text-zinc-500">{row.speciesCode}</td>
            <td className="py-2 text-right tabular-nums">
              {row.totalOnHand.toFixed(1)}
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 3: Replace the placeholder home page**

`web/src/app/page.tsx`:

```tsx
import Link from "next/link";
import { getOnHandBySpecies } from "@/lib/inventory/queries";
import { OnHandTable } from "@/components/OnHandTable";

export default async function Home() {
  const rows = await getOnHandBySpecies();

  return (
    <main className="flex flex-1 flex-col p-8 max-w-3xl mx-auto w-full gap-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold">On-hand inventory</h1>
        <div className="flex gap-3">
          <Link href="/receive" className="rounded bg-black text-white px-4 py-2">
            Receive
          </Link>
          <Link href="/adjust" className="rounded border border-zinc-300 px-4 py-2">
            Adjust
          </Link>
        </div>
      </div>
      <OnHandTable rows={rows} />
    </main>
  );
}
```

- [ ] **Step 4: Manual verification against the local stack**

Ensure `supabase start` is running and Task 6's import has been run at least
once locally (Step 5 of Task 6), so there's real data to look at.

Run (from `web/`): `npm run dev`
Visit `http://localhost:3000`.
Expected: a table listing species names, codes, and on-hand pounds,
sorted alphabetically by species name, with "Receive" and "Adjust" buttons
(not yet functional — Tasks 8 and 9).

- [ ] **Step 5: Commit**

```bash
git add web/src/lib/inventory/queries.ts web/src/components/OnHandTable.tsx web/src/app/page.tsx
git commit -m "Add on-hand-by-species screen"
```

---

### Task 8: Receiving screen

**Files:**
- Create: `web/src/lib/inventory/actions.ts`
- Create: `web/src/components/ReceiveForm.tsx`
- Create: `web/src/app/receive/page.tsx`
- Test: `web/src/lib/inventory/actions.integration.test.ts`

**Interfaces:**
- Consumes: server Supabase client (`web/src/lib/supabase/server.ts`).
- Produces: `receiveLot(input: ReceiveLotInput): Promise<{ lotId: string }>`
  and the `ReceiveLotInput` type, in `web/src/lib/inventory/actions.ts` —
  Task 9 adds `recordAdjustment` to the same file.

- [ ] **Step 1: Write the failing test**

`web/src/lib/inventory/actions.integration.test.ts`:

```ts
import { describe, it, expect, afterEach } from "vitest";
import { createServiceRoleClient } from "@/lib/supabase/service";
import { receiveLot } from "./actions";

describe("receiveLot", () => {
  const supabase = createServiceRoleClient();
  const testCode = `RECVTEST${Date.now()}`;

  afterEach(async () => {
    const { data: species } = await supabase
      .from("species")
      .select("id")
      .eq("code", testCode)
      .maybeSingle();
    if (species) {
      const { data: lots } = await supabase.from("lots").select("id").eq("species_id", species.id);
      for (const lot of lots ?? []) {
        await supabase.from("inventory_transaction").delete().eq("lot_id", lot.id);
      }
      await supabase.from("lots").delete().eq("species_id", species.id);
      await supabase.from("species").delete().eq("id", species.id);
    }
  });

  it("creates a new lot and an app_entered receipt transaction", async () => {
    const result = await receiveLot({
      speciesCode: testCode,
      speciesName: "Receive Test Fish",
      lotCode: "TST-RECVTEST-010126",
      landedAt: "Test Dock",
      landedDate: "2026-01-01",
      weightLbs: 25,
    });

    expect(result.lotId).toBeTruthy();

    const { data: transactions } = await supabase
      .from("inventory_transaction")
      .select("source, transaction_type, quantity")
      .eq("lot_id", result.lotId);

    expect(transactions).toHaveLength(1);
    expect(transactions![0].source).toBe("app_entered");
    expect(transactions![0].transaction_type).toBe("receipt");
    expect(Number(transactions![0].quantity)).toBe(25);
  });

  it("creates a distinct lot on each call, even for the same species", async () => {
    const first = await receiveLot({
      speciesCode: testCode,
      speciesName: "Receive Test Fish",
      lotCode: "TST-RECVTEST-010126",
      landedAt: "Test Dock",
      landedDate: "2026-01-01",
      weightLbs: 25,
    });
    const second = await receiveLot({
      speciesCode: testCode,
      speciesName: "Receive Test Fish",
      lotCode: "TST-RECVTEST-020126",
      landedAt: "Test Dock",
      landedDate: "2026-01-02",
      weightLbs: 10,
    });

    expect(first.lotId).not.toBe(second.lotId);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run (from `web/`): `npx vitest run src/lib/inventory/actions.integration.test.ts`
Expected: FAIL with "Cannot find module './actions'".

- [ ] **Step 3: Write `receiveLot`**

`web/src/lib/inventory/actions.ts`:

```ts
"use server";

import { createClient } from "@/lib/supabase/server";

export interface ReceiveLotInput {
  speciesCode: string;
  speciesName: string;
  lotCode: string | null;
  landedAt: string | null;
  landedDate: string | null;
  weightLbs: number;
}

export async function receiveLot(input: ReceiveLotInput): Promise<{ lotId: string }> {
  const supabase = await createClient();

  const { data: location, error: locationError } = await supabase
    .from("locations")
    .select("id")
    .eq("code", "main")
    .single();
  if (locationError) throw locationError;

  const { data: existingSpecies } = await supabase
    .from("species")
    .select("id")
    .eq("code", input.speciesCode)
    .maybeSingle();

  let speciesId: string;
  if (existingSpecies) {
    speciesId = existingSpecies.id;
  } else {
    const { data: created, error: createError } = await supabase
      .from("species")
      .insert({ code: input.speciesCode, name: input.speciesName })
      .select("id")
      .single();
    if (createError) throw createError;
    speciesId = created.id;
  }

  const { data: lot, error: lotError } = await supabase
    .from("lots")
    .insert({
      species_id: speciesId,
      location_id: location.id,
      lot_code: input.lotCode,
      landed_at: input.landedAt,
      landed_date: input.landedDate,
    })
    .select("id")
    .single();
  if (lotError) throw lotError;

  const { error: transactionError } = await supabase.from("inventory_transaction").insert({
    lot_id: lot.id,
    transaction_type: "receipt",
    quantity: input.weightLbs,
    source: "app_entered",
  });
  if (transactionError) throw transactionError;

  return { lotId: lot.id };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/lib/inventory/actions.integration.test.ts`
Expected: PASS, 2 tests.

- [ ] **Step 5: Build the receiving form**

`web/src/components/ReceiveForm.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { receiveLot } from "@/lib/inventory/actions";

export function ReceiveForm() {
  const router = useRouter();
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(formData: FormData) {
    setSubmitting(true);
    setError(null);
    try {
      await receiveLot({
        speciesCode: String(formData.get("speciesCode")).toUpperCase(),
        speciesName: String(formData.get("speciesName")),
        lotCode: (formData.get("lotCode") as string) || null,
        landedAt: (formData.get("landedAt") as string) || null,
        landedDate: (formData.get("landedDate") as string) || null,
        weightLbs: Number(formData.get("weightLbs")),
      });
      router.push("/");
      router.refresh();
    } catch {
      setError("Could not save this lot. Try again.");
      setSubmitting(false);
    }
  }

  return (
    <form action={handleSubmit} className="flex flex-col gap-4 max-w-md">
      <label className="flex flex-col gap-1">
        Species code
        <input name="speciesCode" required className="border border-zinc-300 rounded px-3 py-2" />
      </label>
      <label className="flex flex-col gap-1">
        Species name
        <input name="speciesName" required className="border border-zinc-300 rounded px-3 py-2" />
      </label>
      <label className="flex flex-col gap-1">
        Lot code (optional)
        <input name="lotCode" className="border border-zinc-300 rounded px-3 py-2" />
      </label>
      <label className="flex flex-col gap-1">
        Landed at (optional)
        <input name="landedAt" className="border border-zinc-300 rounded px-3 py-2" />
      </label>
      <label className="flex flex-col gap-1">
        Landed date (optional)
        <input name="landedDate" type="date" className="border border-zinc-300 rounded px-3 py-2" />
      </label>
      <label className="flex flex-col gap-1">
        Weight (lb)
        <input
          name="weightLbs"
          type="number"
          step="0.1"
          min="0"
          required
          className="border border-zinc-300 rounded px-3 py-2"
        />
      </label>
      {error && <p className="text-red-600">{error}</p>}
      <button
        type="submit"
        disabled={submitting}
        className="rounded bg-black text-white px-4 py-2 disabled:opacity-50"
      >
        {submitting ? "Saving…" : "Receive lot"}
      </button>
    </form>
  );
}
```

- [ ] **Step 6: Create the receive page**

`web/src/app/receive/page.tsx`:

```tsx
import { ReceiveForm } from "@/components/ReceiveForm";

export default function ReceivePage() {
  return (
    <main className="flex flex-1 flex-col p-8 max-w-3xl mx-auto w-full gap-6">
      <h1 className="text-2xl font-semibold">Receive a lot</h1>
      <ReceiveForm />
    </main>
  );
}
```

- [ ] **Step 7: Manual verification**

Run (from `web/`): `npm run dev`
Visit `http://localhost:3000/receive`, submit a test lot (e.g. species code
`MANUALTEST`, weight `12.5`).
Expected: redirected to `/`, and the new species/weight appears in the
on-hand table.

- [ ] **Step 8: Commit**

```bash
git add web/src/lib/inventory/actions.ts web/src/lib/inventory/actions.integration.test.ts web/src/components/ReceiveForm.tsx web/src/app/receive/page.tsx
git commit -m "Add receiving screen"
```

---

### Task 9: Adjustment screen

**Files:**
- Modify: `web/src/lib/inventory/actions.ts` — add `recordAdjustment`
- Modify: `web/src/lib/inventory/queries.ts` — add `getLotsBySpecies`
- Create: `web/src/components/AdjustForm.tsx`
- Create: `web/src/app/adjust/page.tsx`
- Test: `web/src/lib/inventory/actions.integration.test.ts` (extend)

**Interfaces:**
- Consumes: `getOnHandBySpecies` (Task 7), `receiveLot`'s file (Task 8).
- Produces: `recordAdjustment(input: AdjustmentInput): Promise<{ transactionId: string }>`,
  `getLotsBySpecies(speciesId: string): Promise<LotRow[]>`.

- [ ] **Step 1: Write the failing test**

Append to `web/src/lib/inventory/actions.integration.test.ts`:

```ts
import { recordAdjustment } from "./actions";

describe("recordAdjustment", () => {
  const supabase = createServiceRoleClient();
  const testCode = `ADJTEST${Date.now()}`;
  let lotId: string;

  beforeAll(async () => {
    const { lotId: id } = await receiveLot({
      speciesCode: testCode,
      speciesName: "Adjust Test Fish",
      lotCode: "TST-ADJTEST-010126",
      landedAt: "Test Dock",
      landedDate: "2026-01-01",
      weightLbs: 50,
    });
    lotId = id;
  });

  afterEach(async () => {
    // leave transactions -- tests below build on the running total intentionally
  });

  it("records a negative adjustment for a sale", async () => {
    const result = await recordAdjustment({
      lotId,
      quantityChange: -10,
      reason: "sold",
      note: null,
    });
    expect(result.transactionId).toBeTruthy();

    const { data } = await supabase
      .from("current_on_hand")
      .select("on_hand")
      .eq("lot_id", lotId)
      .single();
    expect(Number(data?.on_hand)).toBe(40);
  });

  it("allows an adjustment to take on-hand negative without throwing", async () => {
    await expect(
      recordAdjustment({
        lotId,
        quantityChange: -1000,
        reason: "correction",
        note: "large correction, does not block",
      })
    ).resolves.toBeTruthy();

    const { data } = await supabase
      .from("current_on_hand")
      .select("on_hand")
      .eq("lot_id", lotId)
      .single();
    expect(Number(data?.on_hand)).toBeLessThan(0);
  });
});
```

Add `beforeAll` to the existing `import { describe, it, expect, afterEach } from "vitest";`
line at the top of the file — change it to:

```ts
import { describe, it, expect, afterEach, beforeAll } from "vitest";
```

- [ ] **Step 2: Run the test to verify it fails**

Run (from `web/`): `npx vitest run src/lib/inventory/actions.integration.test.ts`
Expected: FAIL — `recordAdjustment` is not exported yet.

- [ ] **Step 3: Add `recordAdjustment` to `web/src/lib/inventory/actions.ts`**

Append to the existing file (after `receiveLot`):

```ts
export interface AdjustmentInput {
  lotId: string;
  quantityChange: number;
  reason: "sold" | "used" | "spoiled" | "correction";
  note: string | null;
}

// Per CLAUDE.md's "never block the floor" rule: this never validates that
// the resulting on-hand stays non-negative. A discrepancy is a signal to
// reconcile, not an error to reject.
export async function recordAdjustment(
  input: AdjustmentInput
): Promise<{ transactionId: string }> {
  const supabase = await createClient();

  const { data, error } = await supabase
    .from("inventory_transaction")
    .insert({
      lot_id: input.lotId,
      transaction_type: "adjustment",
      quantity: input.quantityChange,
      reason: input.reason,
      note: input.note,
      source: "app_entered",
    })
    .select("id")
    .single();
  if (error) throw error;

  return { transactionId: data.id };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/lib/inventory/actions.integration.test.ts`
Expected: PASS, 4 tests total (2 from Task 8, 2 new).

- [ ] **Step 5: Add `getLotsBySpecies` to `web/src/lib/inventory/queries.ts`**

Append to the existing file:

```ts
export interface LotRow {
  lotId: string;
  lotCode: string | null;
  landedDate: string | null;
  onHand: number;
}

export async function getLotsBySpecies(speciesId: string): Promise<LotRow[]> {
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("current_on_hand")
    .select("lot_id, lot_code, landed_date, on_hand")
    .eq("species_id", speciesId)
    .order("landed_date", { ascending: true, nullsFirst: false });

  if (error) throw error;

  return (data ?? []).map((row) => ({
    lotId: row.lot_id,
    lotCode: row.lot_code,
    landedDate: row.landed_date,
    onHand: Number(row.on_hand),
  }));
}
```

- [ ] **Step 6: Build the adjustment form**

`web/src/components/AdjustForm.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { recordAdjustment } from "@/lib/inventory/actions";
import type { OnHandBySpeciesRow, LotRow } from "@/lib/inventory/queries";

export function AdjustForm({ speciesRows }: { speciesRows: OnHandBySpeciesRow[] }) {
  const router = useRouter();
  const [selectedSpeciesId, setSelectedSpeciesId] = useState("");
  const [lots, setLots] = useState<LotRow[]>([]);
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSpeciesChange(speciesId: string) {
    setSelectedSpeciesId(speciesId);
    if (!speciesId) {
      setLots([]);
      return;
    }
    const response = await fetch(`/api/lots-by-species?speciesId=${speciesId}`);
    const data = await response.json();
    setLots(data.lots);
  }

  async function handleSubmit(formData: FormData) {
    setSubmitting(true);
    setError(null);
    try {
      await recordAdjustment({
        lotId: String(formData.get("lotId")),
        quantityChange: -Math.abs(Number(formData.get("quantity"))),
        reason: formData.get("reason") as "sold" | "used" | "spoiled" | "correction",
        note: (formData.get("note") as string) || null,
      });
      router.push("/");
      router.refresh();
    } catch {
      setError("Could not save this adjustment. Try again.");
      setSubmitting(false);
    }
  }

  return (
    <form action={handleSubmit} className="flex flex-col gap-4 max-w-md">
      <label className="flex flex-col gap-1">
        Species
        <select
          name="speciesId"
          required
          value={selectedSpeciesId}
          onChange={(e) => handleSpeciesChange(e.target.value)}
          className="border border-zinc-300 rounded px-3 py-2"
        >
          <option value="">Select a species…</option>
          {speciesRows.map((row) => (
            <option key={row.speciesId} value={row.speciesId}>
              {row.speciesName} ({row.totalOnHand.toFixed(1)} lb on hand)
            </option>
          ))}
        </select>
      </label>
      <label className="flex flex-col gap-1">
        Lot
        <select name="lotId" required className="border border-zinc-300 rounded px-3 py-2">
          <option value="">Select a lot…</option>
          {lots.map((lot) => (
            <option key={lot.lotId} value={lot.lotId}>
              {lot.lotCode ?? "(no lot code)"} — {lot.onHand.toFixed(1)} lb on hand
            </option>
          ))}
        </select>
      </label>
      <label className="flex flex-col gap-1">
        Quantity leaving (lb)
        <input
          name="quantity"
          type="number"
          step="0.1"
          min="0"
          required
          className="border border-zinc-300 rounded px-3 py-2"
        />
      </label>
      <label className="flex flex-col gap-1">
        Reason
        <select name="reason" required className="border border-zinc-300 rounded px-3 py-2">
          <option value="sold">Sold</option>
          <option value="used">Used</option>
          <option value="spoiled">Spoiled</option>
          <option value="correction">Correction</option>
        </select>
      </label>
      <label className="flex flex-col gap-1">
        Note (optional)
        <input name="note" className="border border-zinc-300 rounded px-3 py-2" />
      </label>
      {error && <p className="text-red-600">{error}</p>}
      <button
        type="submit"
        disabled={submitting}
        className="rounded bg-black text-white px-4 py-2 disabled:opacity-50"
      >
        {submitting ? "Saving…" : "Record adjustment"}
      </button>
    </form>
  );
}
```

- [ ] **Step 7: Add the lots-by-species API route**

`web/src/app/api/lots-by-species/route.ts`:

```ts
import { NextRequest, NextResponse } from "next/server";
import { getLotsBySpecies } from "@/lib/inventory/queries";

export async function GET(request: NextRequest) {
  const speciesId = request.nextUrl.searchParams.get("speciesId");
  if (!speciesId) {
    return NextResponse.json({ lots: [] });
  }
  const lots = await getLotsBySpecies(speciesId);
  return NextResponse.json({ lots });
}
```

- [ ] **Step 8: Create the adjust page**

`web/src/app/adjust/page.tsx`:

```tsx
import { getOnHandBySpecies } from "@/lib/inventory/queries";
import { AdjustForm } from "@/components/AdjustForm";

export default async function AdjustPage() {
  const speciesRows = await getOnHandBySpecies();

  return (
    <main className="flex flex-1 flex-col p-8 max-w-3xl mx-auto w-full gap-6">
      <h1 className="text-2xl font-semibold">Record an adjustment</h1>
      <AdjustForm speciesRows={speciesRows} />
    </main>
  );
}
```

- [ ] **Step 9: Manual verification**

Run (from `web/`): `npm run dev`
Visit `http://localhost:3000/adjust`. Select the species and lot created in
Task 8's manual verification, record an adjustment.
Expected: redirected to `/`, and the on-hand figure for that species has
decreased by the adjusted amount.

- [ ] **Step 10: Commit**

```bash
git add web/src/lib/inventory/actions.ts web/src/lib/inventory/actions.integration.test.ts web/src/lib/inventory/queries.ts web/src/components/AdjustForm.tsx web/src/app/adjust/page.tsx web/src/app/api/lots-by-species/route.ts
git commit -m "Add adjustment screen"
```

---

### Task 10: One-way mirror to the Google Sheet

**This task has manual, external prerequisites that cannot be automated or
verified by an agent working alone.** Complete them before Step 3.

**Files:**
- Create: `supabase/functions/mirror-inventory-to-sheet/index.ts`
- Create: `supabase/migrations/20260824120200_mirror_schedule.sql`

**Interfaces:**
- Consumes: `on_hand_by_species` view (Task 2).
- Produces: a deployed Edge Function, invoked on a `pg_cron` schedule.

- [ ] **Step 1: Manual — create a Google Cloud service account**

1. In the [Google Cloud Console](https://console.cloud.google.com/), create
   or select a project.
2. Enable the **Google Sheets API** for that project (APIs & Services →
   Library → search "Google Sheets API" → Enable).
3. Create a service account (IAM & Admin → Service Accounts → Create Service
   Account). No project-level roles are needed — access is granted by
   sharing the specific sheet, not by IAM.
4. Create a JSON key for that service account (Keys tab → Add Key → Create
   new key → JSON) and download it.
5. Note the service account's email address (looks like
   `fish-house-mirror@<project-id>.iam.gserviceaccount.com`).

- [ ] **Step 2: Manual — share the sheet and create the target tab**

1. In the facility's real Google Sheet, add a new tab named exactly
   `FROM_APP`. Leave it empty — the function writes the header row itself.
2. Share the sheet with the service account's email (Share button → paste
   the email → Editor access).
3. Copy the sheet's ID from its URL (the string between `/d/` and `/edit`).

- [ ] **Step 3: Store secrets for the Edge Function**

Run (from repo root):

```bash
supabase secrets set GOOGLE_SERVICE_ACCOUNT_JSON="$(cat path/to/downloaded-key.json)"
supabase secrets set SHEET_ID="<the sheet ID from step 2>"
```

- [ ] **Step 4: Write the Edge Function**

`supabase/functions/mirror-inventory-to-sheet/index.ts`:

```ts
import { createClient } from "npm:@supabase/supabase-js@2";
import { GoogleAuth } from "npm:google-auth-library@9";

Deno.serve(async () => {
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  const { data: rows, error } = await supabase
    .from("on_hand_by_species")
    .select("species_code, species_name, total_on_hand")
    .order("species_name");

  if (error) {
    return new Response(JSON.stringify({ error: error.message }), { status: 500 });
  }

  const serviceAccount = JSON.parse(Deno.env.get("GOOGLE_SERVICE_ACCOUNT_JSON")!);
  const sheetId = Deno.env.get("SHEET_ID")!;

  const auth = new GoogleAuth({
    credentials: serviceAccount,
    scopes: ["https://www.googleapis.com/auth/spreadsheets"],
  });
  const client = await auth.getClient();
  const accessToken = await client.getAccessToken();

  const values = [
    ["Species code", "Species name", "On hand (lb)", "Mirrored at"],
    ...(rows ?? []).map((row) => [
      row.species_code,
      row.species_name,
      Number(row.total_on_hand).toFixed(1),
      new Date().toISOString(),
    ]),
  ];

  const response = await fetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${sheetId}/values/FROM_APP!A1:D${values.length}?valueInputOption=RAW`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${accessToken.token}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ values }),
    }
  );

  if (!response.ok) {
    const body = await response.text();
    return new Response(JSON.stringify({ error: `Sheets API error: ${body}` }), { status: 502 });
  }

  return new Response(JSON.stringify({ rowsWritten: values.length - 1 }), { status: 200 });
});
```

- [ ] **Step 5: Test locally**

Run (from repo root): `supabase functions serve mirror-inventory-to-sheet`
In a second terminal, run:

```bash
curl -i --request POST 'http://127.0.0.1:54321/functions/v1/mirror-inventory-to-sheet' \
  --header "Authorization: Bearer $(supabase status -o env | grep SERVICE_ROLE | cut -d= -f2)"
```

Expected: `200 OK` with `{"rowsWritten": N}` where N matches the number of
distinct species currently in the local database. This step cannot be
verified by an automated test — it depends on live Google credentials —
so confirm it by hand: open the actual Google Sheet and check the `FROM_APP`
tab was updated.

- [ ] **Step 6: Deploy the function**

Run (from repo root): `supabase functions deploy mirror-inventory-to-sheet`
Expected: deploy succeeds, prints the function's URL on the remote project.

- [ ] **Step 7: Schedule it with `pg_cron`**

`supabase/migrations/20260824120200_mirror_schedule.sql`:

```sql
-- Schedules the sheet mirror. Runs every 15 minutes -- frequent enough that
-- the sheet never looks badly stale, infrequent enough not to hammer the
-- Sheets API. Adjust the schedule string if that cadence turns out wrong
-- once this is in real use.
create extension if not exists pg_cron with schema extensions;
create extension if not exists pg_net with schema extensions;

select vault.create_secret(
  'https://<PROJECT_REF>.supabase.co/functions/v1/mirror-inventory-to-sheet',
  'mirror_function_url'
);
select vault.create_secret(
  '<SERVICE_ROLE_KEY>',
  'mirror_function_service_role_key'
);

select cron.schedule(
  'mirror-inventory-to-sheet',
  '*/15 * * * *',
  $$
  select net.http_post(
    url := (select decrypted_secret from vault.decrypted_secrets where name = 'mirror_function_url'),
    headers := jsonb_build_object(
      'Authorization', 'Bearer ' || (select decrypted_secret from vault.decrypted_secrets where name = 'mirror_function_service_role_key'),
      'Content-Type', 'application/json'
    )
  );
  $$
);
```

Before applying: replace `<PROJECT_REF>` with the remote project's ref (from
the Supabase dashboard URL) and `<SERVICE_ROLE_KEY>` with the remote
project's service role key (Dashboard → Project Settings → API). This
migration is written for the **remote** project, not local — `pg_cron`
scheduling a real HTTP call to a deployed function has no equivalent local
smoke test. Apply it with `supabase db push` once the placeholders are
filled in, not `supabase db reset`.

- [ ] **Step 8: Verify on the remote project**

Wait 15 minutes after deploying, or trigger the cron job manually via:

```sql
select cron.schedule_in_seconds('mirror-inventory-to-sheet-once', 5, $$ ... same body as above ... $$);
```

Expected: the `FROM_APP` tab in the real Google Sheet updates with current
on-hand data.

- [ ] **Step 9: Commit**

```bash
git add supabase/functions/mirror-inventory-to-sheet/index.ts supabase/migrations/20260824120200_mirror_schedule.sql
git commit -m "Add one-way inventory mirror to the Google Sheet"
```

---

## Self-Review

**Spec coverage:** every "In scope for v1" bullet from the design spec has a
task — ledger (Task 2), receiving (Task 8), adjustment (Task 9), UI (Tasks
7–9), importer (Tasks 4–6), one-way mirror (Task 10). RLS/auth wasn't called
out as its own spec bullet but is required by CLAUDE.md's "RLS is still
used" rule — covered by Task 3.

**Placeholder scan:** the two literal placeholders (`<PROJECT_REF>`,
`<SERVICE_ROLE_KEY>` in Task 10, Step 7) are not code-logic gaps — they're
values that only exist once a real Supabase project is provisioned, which is
outside this repo. Every other step has complete, concrete code.

**Type consistency:** `ParsedLotRow` (Task 5) matches its usage in Task 6.
`ReceiveLotInput`/`AdjustmentInput` (Tasks 8/9) match the form submissions in
`ReceiveForm.tsx`/`AdjustForm.tsx`. `OnHandBySpeciesRow`/`LotRow` (Task 7/9)
match their consumers in `AdjustForm.tsx`. `createServiceRoleClient` (Task 2)
is used consistently by name across Tasks 3, 6.

**Task 10's manual dependency:** flagged clearly at the top of the task and
in Steps 1, 2, 5, 7, 8 — this is inherent to integrating with an external
Google Cloud project and cannot be removed, only made as explicit as
possible, which it is.
