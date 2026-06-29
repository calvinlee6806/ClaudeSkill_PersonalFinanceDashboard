# ClaudeSkill — Personal Finance Dashboard

AI-assisted skills that turn messy bank data into a clean finance database, then present it as a
dashboard. Drop in a statement, get clean categorized rows, see where your money went.

## Goal

Give one trustworthy answer to "where did my money go — this month, on this trip, on non-routine
things?" without manually cleaning data every month.

## The problem it solves

Spending data is scattered and messy: every bank exports a different CSV layout, some records
only exist as screenshots or PDFs, and merchant names are cryptic ("UBER *TRIP 02-14 AMST").
There is no single place that adds it all up. Doing it by hand is tedious and inconsistent.

The fix is to **separate interpretation from facts, and data from presentation**:

- An **LLM** handles interpretation — reads any bank's columns, cleans merchant names, suggests
  categories — so the rules never need to know each bank's format.
- **Deterministic code** handles facts — deduplication, travel flagging by trip dates,
  notable-spend flagging by threshold — so results are reproducible and testable.
- A **human reviews** low-confidence rows before they land, keeping accuracy high.
- One **Excel database** is the single source of truth; any number of dashboards read from it.

## How it works

```
Layer 1 · DATA          finance-pipeline (skill)
   bank files → AI clean & categorize → human review → FinanceDB.xlsx   (single source of truth)
        │
        ▼
Layer 2 · PRESENTATION  finance-dashboard-html (skill)  →  End Product A: portable HTML snapshot
                        FinanceDashboardApp (.NET 10)    →  End Product B: live app (reads Excel on load)
```

Both end products read the **same** database, so they never disagree. A is portable and offline
(re-generate after data changes); B is always-current (just refresh).

## What's in here

| Skill | What it does | Output |
| :--- | :--- | :--- |
| **`finance-pipeline`** | Ingest CSV / Excel / screenshots, clean & categorize with AI, pause for human review of low-confidence rows, then write deduplicated rows into the Excel database. | Updated `FinanceDB.xlsx` |
| **`finance-dashboard-html`** | Read the database and generate one self-contained HTML dashboard (charts by year/month/week, category & tag breakdowns, trip spend, sortable table). | `Finance_Dashboard.html` |

The two skills are deliberately separate: `finance-pipeline` owns the **truth**; presentation
skills only read from it. `FinanceDashboardApp` (End Product B) is a separate .NET project that
serves a live dashboard from the same database.

## Layout

```
ClaudeSkill_PersonalFinanceDashboard/
├── README.md                         ← you are here
├── finance-pipeline/
│   ├── SKILL.md
│   └── references/                   schema.md · rules.md · prompts.md
└── finance-dashboard-html/
    ├── SKILL.md
    ├── scripts/build_dashboard.py
    └── assets/dashboard_template.html
```

## Quick start

1. **Install the skills** via Settings → Capabilities (these folders are the packages).
2. **Add data:** drop a bank file in the input folder and ask Claude to process it. It cleans,
   categorizes, asks about anything uncertain, and appends confirmed rows to `FinanceDB.xlsx`.
3. **See it:** ask Claude to build the HTML dashboard, or run `FinanceDashboardApp` (`dotnet run`)
   for a live view.

## Conventions

- All code, comments, and prompts are in **English**.
- Schema and deterministic rules live in `finance-pipeline/references/` — read those before
  changing how rows are written.
- Presentation skills are **read-only** — they never modify the database.

## Requirements

- Python 3.9+ with `openpyxl` for the HTML dashboard.
- A workbook following `finance-pipeline/references/schema.md` (start from
  `FinanceDB_template.xlsx`).
- .NET 10 SDK + ClosedXML (restored automatically) for the optional live app.
