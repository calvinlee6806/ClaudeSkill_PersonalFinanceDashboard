# Check Report Templates

Two reusable report formats for the pipeline's confirmation checkpoints. Reuse these verbatim so
every import looks like the same app screen. Replace `<...>` placeholders; drop rows that don't
apply (e.g. no unknown currency this batch).

---

## Checkpoint 1 — Intake review (BEFORE writing to the database)

Show this after aligning and categorizing, then STOP and wait for the user.

```
INTAKE REVIEW — <N> file(s): <Bank A>, <Bank B>, <Bank C>
Parsed <T> transactions · <earliest> → <latest>

Please confirm before I write:

❓ Unknown currency
   - <CUR> (<count> rows) — which GBP rate should I use?

❓ New categories / merchants (not yet in Rules)
   - "<RawMerchant>" → I'd categorize as <MainCategory>/<SubCategory>. OK to learn?

⚠ Low-confidence / ambiguous (AI_Confidence < 0.7)
   - <date> · <merchant> · <amount> — <why it's unclear>

✔ Everything else looks clear (<K> rows).
Reply with any fixes, or "go ahead" to write.
```

If nothing is uncertain, still send a short version and wait:

```
INTAKE REVIEW — <N> file(s): <sources>
Parsed <T> transactions · <range>. Nothing ambiguous in this batch.
Say "go ahead" and I'll write them in.
```

---

## Checkpoint 2 — Commit confirmation (AFTER writing to the database)

Show this once rows are appended (post dedup), then ask the user to confirm.

```
✅ IMPORT COMPLETE — please confirm
Sources: <Bank A>, <Bank B>, <Bank C> — <N> file(s)
Added: <M> new transactions (skipped <K> duplicates)
Total spend this batch: £<X>
Date range: <earliest> → <latest>
Flagged for review: <R> row(s)

Confirm to finalize (I'll archive the source files), or tell me what to fix.
```

Also say it in one natural sentence, e.g.:
> *"This batch came from Revolut, Taishin and Sino (3 sources), added 42 new transactions
> (skipped 5 duplicates), total spend £1,234.56."*

Only after the user confirms: update `Rules` with newly learned merchants and move the source
files to `archive/`.

---

## Worked example

```
✅ IMPORT COMPLETE — please confirm
Sources: Revolut (GBP), Taishin Richart, Sino — 3 files
Added: 42 new transactions (skipped 5 duplicates)
Total spend this batch: £1,234.56
Date range: 2026-06-01 → 2026-06-24
Flagged for review: 3 rows

Confirm to finalize, or tell me what to fix.
```

## Notes

- **Total spend** = sum of positive `BaseAmount` in this batch (exclude negative cashback/refunds;
  you may mention those separately).
- **Duplicates** are detected by `TransactionID` (see `rules.md`), so overlapping or re-dropped
  files never double-import.
- Keep both reports short and scannable — they are status screens, not essays.
