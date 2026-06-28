# ClaudeSkill_PersonalFinanceDashboard
A small, AI-assisted pipeline that turns messy bank data into a clean finance database, then presents it as a dashboard. It is delivered as two Claude skills plus one optional app.

# Personal Finance Pipeline — Purpose & Instructions

## What this is

A small, AI-assisted pipeline that turns messy bank data into a clean finance database, then
presents it as a dashboard. It is delivered as two Claude **skills** plus one optional **app**.

## The problem it solves

Personal spending data is scattered and messy: every bank exports a different CSV layout, some
records only exist as screenshots or PDFs, merchant names are cryptic ("UBER *TRIP 02-14 AMST"),
and there is no single place that answers "where did my money go this month / on this trip / on
non-routine things?". Manually cleaning and categorizing this every month is tedious and
inconsistent.

This project fixes that by **separating interpretation from facts, and data from presentation**:

- An **LLM** handles the messy interpretation (reading any bank's columns, cleaning merchant
  names, suggesting categories) — so the rules never need to know each bank's format.
- **Deterministic code** handles the facts (deduplication, travel flagging by trip dates,
  notable-spend flagging by threshold) — so results are reproducible and testable.
- A **human reviews** low-confidence rows before they land — accuracy stays high.
- One **Excel database** is the single source of truth; any number of dashboards read from it.

## Architecture

```
Layer 1 · DATA          finance-pipeline (skill)
   bank files → AI clean & categorize → human review → FinanceDB.xlsx   (single source of truth)
        │
        ▼
Layer 2 · PRESENTATION  finance-dashboard-html (skill)  →  End Product A: portable HTML snapshot
                        FinanceDashboardApp (.NET)       →  End Product B: live app (reads Excel on each load)
```

Both end products read the **same** database, so they never disagree. A is portable and offline
(re-generate after data changes); B is always-current (just refresh).

## Repository contents

| Path | What it is |
| :--- | :--- |
| `finance-pipeline/` | Skill: ingest → clean/categorize → human review → update the Excel DB. |
| `finance-dashboard-html/` | Skill: read the DB → generate one self-contained HTML dashboard. |
| `../FinanceDashboardApp/` | Optional .NET 10 app: serves a live dashboard from the same DB. |

## Quick start

1. **Install the skills** via Settings → Capabilities (the folders here are the packages).
2. **Add data:** drop a bank file in the input folder and ask Claude to process it. It cleans,
   categorizes, asks you about anything uncertain, and appends confirmed rows to `FinanceDB.xlsx`.
3. **See it:** ask Claude to build the HTML dashboard, or run `FinanceDashboardApp` (`dotnet run`)
   for a live view.

## Conventions

- All code, comments, and prompts are in **English**.
- The database schema and the deterministic rules are documented in
  `finance-pipeline/references/`. Read those before changing how rows are written.
- Presentation skills are **read-only** — they never modify the database.

## Requirements

- Python 3.9+ with `openpyxl` for the HTML dashboard.
- .NET 10 SDK + ClosedXML (restored automatically) for the optional app.
