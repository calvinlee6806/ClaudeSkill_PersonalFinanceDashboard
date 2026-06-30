# Deterministic Rules

These run on the *standardized* rows (after AI alignment), never on the raw bank file. They are
pure functions of the data, so they are reproducible and easy to test.

## 1. Dedup hash — `TransactionID`

```
TransactionID = first 12 hex chars of  md5( TransactionDate_ISO + "|" + OriginalAmount + "|" + RawMerchant )
```

- Use the ISO date (`YYYY-MM-DD`), the raw amount as written, and the raw (uncleaned) merchant
  string, so the hash is stable regardless of later cleaning.
- On insert, skip any row whose `TransactionID` already exists in the `Database`. This makes
  re-running the same file a no-op (idempotent).

## 2. `IsTravel` — by date range, NOT by currency

For each transaction, `IsTravel = TRUE` if its `TransactionDate` falls within any
`[StartDate, EndDate]` range in the `Trips` sheet (inclusive on both ends).

When it matches a trip, also append the trip name to `CustomTags` (e.g. `Greece`). Currency is
never used to infer travel.

## 3. `IsNotable` — two layers

Surface non-routine spending automatically. `IsNotable = TRUE` if **any** of:

- **(a) Category layer:** `MainCategory` is NOT in `Config.RoutineCategories`
  (default routine list: Food, Transport, Groceries, Bills, Rent). This catches clothes,
  electronics, furniture, etc.
- **(b) Amount layer:** `BaseAmount >= Config.NotableThreshold` (default 100), catching large
  one-offs even inside a routine category.
- **(c) Travel:** `IsTravel = TRUE` is also treated as notable.

The routine list and threshold live in `Config` and are user-tunable — always read them from the
workbook rather than hard-coding.

## 4. Normalization

- **Dates** → real date values (`YYYY-MM-DD`), not strings.
- **Amounts** → numbers, not text; strip currency symbols and thousands separators.
- **Currency** → uppercase ISO code (GBP, USD, TWD…).

## 5. `NeedsReview`

`NeedsReview = TRUE` when `AI_Confidence < 0.7` (or the threshold in `Config`). These rows are
the ones to raise with the user before/after insertion so they can confirm or fix them in Excel.
