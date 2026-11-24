# SubLedger — Subscription & Expense Tracker

**SubLedger** is a full-stack personal finance and recurring spend tracker built with **Next.js (App Router)**, **TypeScript**, **Tailwind CSS**, **Prisma**, and **SQLite**. It unifies recurring subscriptions and one-time expenses into a single normalized monthly spend ledger, pairs them with configurable category budgets and threshold alerts, surfaces upcoming/overdue renewal reminders with one-click cycle advancement, and exports filtered views to formula-safe CSV.

---

## 1. Overview & Problem Solved

Most personal tracking tools either track one-off transactions (losing sight of recurring SaaS/utility commitments) or list subscriptions in isolation (missing how discretionary spending pushes a category over budget). **SubLedger** solves both in one coherent workflow:

- **Normalized Monthly Rollups:** Converts `WEEKLY`, `MONTHLY`, and `YEARLY` subscriptions into an apples-to-apples monthly run-rate (`amountCents * 52 / 12`, `amountCents`, `amountCents / 12`), combined with one-time expenses for any selected month.
- **Category Budget Thresholds:** Tracks recurring + one-time spend per category against configurable monthly limits and warning thresholds (e.g. `80%`), surfacing `OK`, `WARNING`, `EXCEEDED`, and `UNBUDGETED` states.
- **Renewal Alert Window & One-Click Renewal:** Flags active subscriptions that are overdue or renewing within 14 days, and lets users advance `nextRenewalDate` by one billing interval (with end-of-month clamping) or pause unused plans.
- **Full Ledger CRUD & Safe CSV Export:** Supports search, multi-facet filtering, sorting, inline form validation, demo data reset, and RFC 4180 CSV export with spreadsheet formula-injection neutralization.

---

## 2. Architecture & Key Design Decisions

### Project Structure

```text
subledger/
├── prisma/
│   ├── schema.prisma          # Relational SQLite schema (Entry, CategoryBudget)
│   └── seed.ts                # CLI seed entry point
├── src/
│   ├── app/
│   │   ├── page.tsx           # Server Component initial render (zero blank-screen flash)
│   │   ├── layout.tsx         # Root layout and typography
│   │   └── api/
│   │       ├── health/        # DB connectivity check
│   │       ├── dashboard/     # Aggregated monthly KPIs, budgets, trend, renewal alerts
│   │       ├── entries/       # CRUD + filtering/sorting + POST /[id]/renew action
│   │       ├── budgets/       # Category budget CRUD with uniqueness conflict handling
│   │       ├── export/        # Filtered CSV download endpoint
│   │       └── seed/          # Reset/restore relative-dated demo dataset
│   ├── components/            # Modular UI components (charts, alerts, modals, table, toasts)
│   └── lib/
│       ├── db.ts              # Prisma client singleton + race-safe auto-seed guard
│       ├── finance.ts         # Pure domain logic (amortization, budgets, renewals, CSV)
│       ├── queries.ts         # Prisma filter/sort query builders shared by /entries & /export
│       ├── seed-data.ts       # Relative-date sample dataset builder
│       ├── serializers.ts     # Prisma-to-DTO serialization
│       ├── types.ts           # Shared TypeScript interfaces & domain constants
│       └── validation.ts      # Zod schemas, cross-field superRefine rules, error formatter
└── tests/
    ├── finance.test.ts        # Unit tests for pure domain logic
    └── api.test.ts            # Integration tests against SQLite + Next.js route handlers
```

### Key Design Decisions & Trade-Offs

1. **Integer Cents (`amountCents: Int`) Instead of Floating-Point Dollars:**
   - *Why:* Eliminates IEEE-754 rounding drift (`0.1 + 0.2 !== 0.3`) in rollups, budget comparisons, and CSV exports.
   - *Trade-off:* Requires parsing/formatting at UI and API boundaries; scales cleanly to multi-currency decimal-exponent tables if needed later.
2. **Unified `Entry` Model (`SUBSCRIPTION` | `EXPENSE`) vs. Separate Tables:**
   - *Why:* Allows unified search, sorting, category rollups, and CSV export without expensive cross-table `UNION` pagination, while Zod `.superRefine()` enforces strict per-kind invariants (`EXPENSE` requires `ONE_TIME` + `PAID` + `null` renewal; `SUBSCRIPTION` requires `WEEKLY`/`MONTHLY`/`YEARLY` + renewal date when active/paused).
   - *Trade-off:* `nextRenewalDate` is nullable in the schema; at much larger scale with subscription-specific payment history, splitting out an immutable `BillingEvent` ledger table would be a natural evolution.
3. **Pure Functional Domain Layer (`src/lib/finance.ts`) Decoupled from Prisma:**
   - *Why:* All financial math (monthly normalization, date clamping, budget thresholds, renewal urgency, CSV escaping) lives in pure TypeScript functions, making unit testing fast and deterministic without mocking a database.
4. **SQLite + Prisma with Zero External Cloud Keys:**
   - *Why:* Runs locally in seconds (`file:./dev.db`) with auto-seeding on first load and an explicit `Reset Demo` endpoint. Swapping to PostgreSQL requires changing only `provider` and `DATABASE_URL`.
5. **What Would Change at Production Scale:**
   - Add user authentication (`userId` foreign keys + row-level authorization checks on every query).
   - Move dashboard rollups and 6-month trend aggregation into indexed SQL `GROUP BY` / materialized monthly snapshots instead of loading all entries in memory.
   - Introduce an immutable `RenewalHistory` / `TransactionLog` table to track historical subscription price changes and actual charges over time.
   - Add background cron/webhook ingestion (e.g. Stripe/Plaid) with idempotency keys rather than manual "Mark Renewed" clicks.

---

## 3. Step-by-Step Local Setup

### Prerequisites
- **Node.js** `>= 20` (tested on Node `v26`) and **npm**

### Install, Seed & Run

```bash
# 1. Install dependencies
npm install

# 2. Configure local SQLite environment variable
cp .env.example .env
# (On Windows PowerShell: Copy-Item .env.example .env)

# 3. Generate Prisma Client, push SQLite schema, and seed initial data
npm run db:setup

# 4. Start the development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

### Run Automated Tests & Production Build

```bash
# Run unit + API integration tests
npm test

# Verify TypeScript + production Next.js build
npm run build
```

> **Windows PowerShell note:** If PowerShell blocks `npm.ps1` due to execution policy, run `npm.cmd install`, `npm.cmd run db:setup`, `npm.cmd test`, and `npm.cmd run dev`.

---

## 4. Likely Interview Questions & Talking Points

1. **How did you prevent floating-point money bugs?**
   - Monetary values are converted to integer cents on input (`parseDollarsToCents`) and persisted as SQLite `INTEGER` (`amountCents`, `monthlyLimitCents`), only formatting back to currency strings at presentation/export time.
2. **How do you handle date/timezone edge cases on recurring billing?**
   - All billing dates (`occurredAt`, `nextRenewalDate`) are normalized to UTC midnight via `toUtcDateOnly()`. Without this, a user in UTC-5 vs. UTC+5:30 can experience off-by-one-day drift on renewal countdowns. Additionally, `advanceRenewalDate()` clamps month-end dates (`2026-01-31` -> `2026-02-28` instead of overflowing to March 3).
3. **How do you guarantee data integrity on partial `PATCH` updates?**
   - `PATCH /api/entries/[id]` fetches the existing row, merges the partial payload with persisted values, and runs the full `validateMergedEntry` (`superRefine`) schema on the merged result. That prevents a partial update from creating an invalid state (e.g., changing `kind` to `EXPENSE` while leaving `billingCycle` as `MONTHLY`).
4. **What security edge case did you handle in CSV export?**
   - `escapeCsvCell()` follows RFC 4180 quoting and prepends a leading `'` to cells starting with `=`, `+`, `-`, `@`, `\t`, or `\r` to mitigate CSV formula injection when opened in Excel or Google Sheets.
5. **How does the app behave on first run without manual setup?**
   - `ensureSeeded()` in `src/lib/db.ts` checks if both tables are empty on first request and seeds relative-dated demo records inside a Prisma transaction guarded by a shared in-memory promise lock to avoid race conditions under concurrent requests.
