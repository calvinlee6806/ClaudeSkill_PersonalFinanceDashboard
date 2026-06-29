# Target Schema — `Database` sheet

Every transaction is transformed into this schema before insertion. Columns in this exact order.

| Column | Type | Description | Required |
| :--- | :--- | :--- | :--- |
| `TransactionID` | String | Hash of `TransactionDate` + `OriginalAmount` + `RawMerchant`. Dedup key. | ✅ |
| `TransactionDate` | Date | Date of the transaction. | ✅ |
| `OriginalAmount` | Number | Exact amount charged, in the original currency. Positive = spend, negative = refund/cashback/income. | ✅ |
| `Currency` | String | GBP, USD, JPY, TWD, etc. | ✅ |
| `BaseAmount` | Number | Amount converted to GBP (informational only). | |
| `Account` | String | Card / account the charge came from. | |
| `RawMerchant` | String | Original messy bank description. | ✅ |
| `MerchantName` | String | AI-cleaned merchant name. | ✅ |
| `MainCategory` | String | Food, Groceries, Transport, Bills, Rent, Clothing, Shopping, Entertainment, Subscriptions, Home, Travel, Fees, Cashback, Other. | ✅ |
| `SubCategory` | String | e.g. Supermarket, Taxi, Cafe, Bar. | |
| `Ownership` | String | "Personal" or "Shared". | |
| `IsTravel` | Boolean | TRUE if `TransactionDate` falls in a `Trips` range. | |
| `IsNotable` | Boolean | TRUE per the two-layer rule (see `rules.md`). | |
| `CustomTags` | String | Comma-separated tags (including trip name). | |
| `AI_Confidence` | Float | 0.0–1.0. Flag if < 0.7. | ✅ |
| `NeedsReview` | Boolean | TRUE if `AI_Confidence` < threshold — user should check in Excel. | |
| `SourceFile` | String | Original file the row came from. | |
| `ImportedAt` | DateTime | When the row was added. | |

## Notes

- **Base currency is GBP.** Keep the original `Currency`/`OriginalAmount` untouched; `BaseAmount`
  is a convenience conversion and is NOT used to infer anything (especially not travel — the
  user is a UK-based Taiwanese who sometimes uses TW cards at home, so currency ≠ location).
- **Sign convention:** spending is positive. Cashback, refunds, and incoming amounts are
  negative, so a presentation layer can separate "spend" (`> 0`) from "returned" (`< 0`).
- Start a new database from `0_Config/templates/FinanceDB_template.xlsx`.
