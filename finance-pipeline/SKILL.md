---
name: finance-pipeline
description: Ingest and clean personal-finance transactions into a single Excel database. Use this whenever the user drops a bank statement, CSV, Excel export, or a screenshot/PDF of transactions and wants them cleaned, categorized, and added to their finance database (FinanceDB.xlsx) — even if they just say "process this statement", "add these expenses", "import my Revolut export", or "log these receipts". Handles any bank's layout via AI column alignment, then applies deterministic rules (dedup, travel flagging, notable-spend flagging). Runs two confirmation checkpoints like an app interface — an intake review that raises questions (unknown currency, new categories/merchants) before writing, and a commit summary ("from X, Y, Z — N sources, M new rows, total £…") for sign-off after writing. This is the DATA layer; pair it with a presentation skill (e.g. finance-dashboard-html) to visualize the result.
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

Use the ready-made report formats in `references/check_report.md` for both checkpoints below so
every import looks like the same app screen.

5. **Checkpoint 1 — Intake review (raise questions BEFORE writing).** Set `AI_Confidence` and
   `NeedsReview = TRUE` for each row, then show a compact intake report and *stop* to ask about
   anything uncertain. Treat this like an app's review screen — do not write to the database
   until the user has answered. Always surface, at minimum:
   - **Unknown currency** — a currency you have not converted before, or one without a clear
     GBP rate. Ask which rate to use.
   - **New categories / merchants** — merchants not in `Rules` and any category you are guessing.
     Confirm before learning them.
   - **Ambiguous or low-confidence rows** — `AI_Confidence < 0.7`, surprising categories, odd
     dates, or possible duplicates.

   If nothing is uncertain, say so explicitly ("Nothing ambiguous in this batch") and still wait
   for a quick go-ahead. This human-in-the-loop step is the point of the skill — never skip it
   to save a turn.

6. **Insert (idempotent).** After the user resolves Checkpoint 1, dedup against existing
   `TransactionID`s, append only new rows to the `Database` table, and never modify existing
   rows. Re-running the same file must be a no-op. Because `input/` accumulates files over time,
   this ID-based dedup is what prevents the same transaction being imported twice; count how many
   incoming rows were new vs. skipped as duplicates.

7. **Checkpoint 2 — Commit confirmation (summary + sign-off).** Once the rows are written, show a
   confirmation report and ask the user to confirm, like a real app's "import complete" screen.
   Use this exact shape:

   > **Import complete — please confirm**
   > Sources: `<Bank A>`, `<Bank B>`, `<Bank C>` — `<N>` file(s)
   > Added: `<M>` new transactions (skipped `<K>` duplicates)
   > Total spend this batch: `£<X>`
   > Date range: `<earliest>` → `<latest>`
   > Flagged for review: `<R>` row(s)
   >
   > Confirm to finalize, or tell me what to fix.

   Phrase it naturally too, e.g. *"This batch came from Revolut, Taishin and Sino (3 sources),
   added 42 new transactions, total spend £1,234.56."* If the user spots a problem, fix the
   affected rows (they are addressable by `TransactionID`) before moving on.

8. **Learn & file.** After the user confirms, add any newly confirmed merchant→category mappings
   to `Rules`, move the processed source file into `archive/`, and note which file was archived.

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
