---
name: Finance-UpdateDashboard
description: Generate a self-contained HTML dashboard from a personal-finance Excel database (FinanceDB.xlsx). Use this whenever the user wants to see, visualize, refresh, or share their finances as a dashboard or report — phrases like "build my finance dashboard", "refresh the dashboard", "show my spending", "make a chart of my expenses by month", or "I updated the database, regenerate the view". Produces one portable .html file (charts by year/month/week, category and tag breakdowns, trip spend, sortable transaction table) that opens offline in any browser with no install. This is a PRESENTATION layer for the Finance-ImportStatement database — it reads, never writes, the data.
---

# Finance Dashboard (HTML)

Read the finance database and produce **one** self-contained HTML file: all data is embedded,
all charts are drawn in vanilla JS/SVG, so it opens offline in any browser with nothing to
install. This is *End Product A* — portable and shareable. It is a snapshot: it reflects the
database at the moment it was built, so re-run this skill after the database changes.

This skill only reads the database. The source of truth stays in `FinanceDB.xlsx`, owned by the
`Finance-ImportStatement` skill.

## How to build

The work is done by a bundled script — don't hand-write HTML or re-implement the charts.

```bash
python scripts/build_dashboard.py <path/to/FinanceDB.xlsx> [output.html]
```

It reads the `Database` and `Trips` sheets, embeds them into `assets/dashboard_template.html`,
and writes the final file (default `Finance_Dashboard.html`). Requires `openpyxl`
(`pip install openpyxl`).

Steps:

1. Find the database. Default to the Live DB (`2_Live/output/FinanceDB.xlsx`); use the Test DB
   (`1_Test/output/FinanceDB_TEST.xlsx`) if the user is working in Test. If the workbook is open
   in Excel (a `.~lock.*` file sits next to it), ask the user to close it first.
2. Run the script, pointing the output at where the user wants it.
3. Present the resulting `.html` file so the user can open it.
4. Sanity-check: confirm the total spend and the transaction count match the database before
   handing it over.

## What the dashboard shows

- **KPI cards** — Total Spend (positive amounts only), Avg/Month, Travel, Notable (≥ threshold),
  and Cashback/incoming (negative amounts) shown separately.
- **Spending over time** — switchable Month / Week / Year.
- **Breakdowns** — by Category, by tag (Travel / Notable / Routine), Top Merchants, and Spend
  by Trip (using the `Trips` date ranges).
- **Transactions** — filterable (account, category) and sortable table.

Spend totals count only positive amounts; cashback/refunds (negative) are reported in their own
card so they don't quietly shrink the spend figure.

## Customizing

- Colors, fonts, and layout live in the `<style>` block of `assets/dashboard_template.html`.
- The data shape and all chart logic live in the `<script>` block; the script injects the data
  into the `__DATA__` placeholder. To add a chart, extend the template's script — the build
  script does not need changing.

## Relationship to other end products

This produces a static HTML snapshot. If the user instead wants an always-current view with no
regeneration step, that is *End Product B* — a separate C# app that reads the same
`FinanceDB.xlsx` live on launch. Both read the same database, so they never disagree.
