---
name: finance-pipeline
description: Ingest and clean personal-finance transactions into a single Excel database. Use this whenever the user drops a bank statement, CSV, Excel export, or a screenshot/PDF of transactions and wants them cleaned, categorized, and added to their finance database (FinanceDB.xlsx) — even if they just say "process this statement", "add these expenses", "import my Revolut export", or "log these receipts". Handles any bank's layout via AI column alignment, then applies deterministic rules (dedup, travel flagging, notable-spend flagging) and pauses for human review of uncertain rows. This is the DATA layer; pair it with a presentation skill (e.g. finance-dashboard-html) to visualize the result.
---

# Finance Pipeline

Turn messy, bank-specific transaction files into clean, deduplicated rows in one Excel
database that is the single source of truth. The database stays human-editable; this skill
only ever *appends* confirmed rows and never overwrites the user's edits.

Why split this way: bank formats vary wildly, so an LLM handles the *interpretation* (reading
each bank's columns), and simple deterministic code handles the *facts* (dates, dedup, travel,
notable). Keeping those apart means the rules never have to know about bank quirks, and the
results are reproducible.

## The database

`FinanceDB.xlsx` is the source of truth. The `Database` sheet is an Excel Table so new rows are
picked up automatically. Supporting sheets the user tunes by hand:

- **Trips** — one row per trip (name, start, end, optional location). Drives travel flagging.
- **Config** — base currency (GBP), `NotableThreshold` (default 100), `RoutineCategories`.
- **Rules** — merchant→category lookup. Grows every time the user confirms a categorization,
  so the system gets more consistent and cheaper over time.

The full column schema is in `references/schema.md`. Read it before writing any rows.

## Workflow

Work through these phases. Read the linked reference only when you reach that phase.

1. **Ingest.** Locate the dropped file in the relevant `input/` folder (Test vs Live). Accept
   CSV, Excel, or images/PDF of statements and receipts.

2. **Align (AI).** Read the raw file and map its columns to the target schema. For images,
   transcribe the visible transactions. Use the alignment prompt in `references/prompts.md`.
   If you have seen this bank before, reuse its saved column mapping for speed and consistency.

3. **Clean & categorize (AI).** Tidy merchant strings ("UBER *TRIP 02-14 AMST" → "Uber"),
   then assign `MainCategory`/`SubCategory`. Check the `Rules` sheet first; only use AI
   judgment for merchants not already in `Rules`. See `references/prompts.md`.

4. **Apply deterministic rules.** Run these on the standardized rows, never on the raw file:
   dedup hash (`TransactionID`), `IsTravel` by Trips date range, `IsNotable` by the two-layer
   rule, currency/number normalization. Exact definitions in `references/rules.md`.

5. **Human review.** Set `NeedsReview = TRUE` and `AI_Confidence` for each row. Before writing,
   show the user a compact preview and explicitly ask about anything low-confidence or
   ambiguous (unknown merchant, surprising category, possible duplicate). Let the user correct
   it. This human-in-the-loop step is the point of the skill — do not skip it to save a turn.

6. **Insert (idempotent).** Dedup against existing `TransactionID`s, append only new rows to the
   `Database` table, and never modify existing rows. Re-running the same file must be a no-op.

7. **Learn & file.** Add any newly confirmed merchant→category mappings to `Rules`. Move the
   processed source file into `archive/`. Tell the user how many rows were added, how many were
   flagged for review, and which file was archived.

## Test vs Live

- `1_Test/` — throwaway data; working DB is `1_Test/output/FinanceDB_TEST.xlsx`.
- `2_Live/` — real money; DB is `2_Live/output/FinanceDB.xlsx`.

Default to Test unless the user clearly means Live. Never touch a workbook while it is open in
Excel (a `.~lock.*` file next to it means it is open) — ask the user to close it first, or the
write may fail or corrupt.

## After processing

Once the database is updated, offer to refresh the dashboard. If the `finance-dashboard-html`
skill is available, that regenerates the HTML view; a separate C# app (if the user built one)
picks up the change automatically on next launch.
