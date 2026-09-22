# Power Query Validation Patterns

Power Query is not only a data transformation tool.

It can also be used as an important validation layer between raw source data and the Power BI semantic model.

A robust process should make transformations:

- visible
- reproducible
- explainable
- reversible where possible
- easy to validate

> **Transform data only after understanding what the transformation changes and why it is necessary.**

---

## 1. Recommended Query Structure

A useful Power Query architecture is:

```text
Raw Source
    ↓
Staging
    ↓
Validation
    ↓
Clean / Standardized
    ↓
Model
```

The exact architecture depends on the project, but separating raw ingestion from business transformations makes troubleshooting easier.

---

## 2. Preserve the Raw Source

Whenever practical, keep a raw or minimally transformed query.

For example:

```text
Orders_Raw
Orders_Staging
Orders_Final
```

The raw query provides a reference when later transformations create unexpected results.

It helps answer:

```text
Was the value already wrong in the source?

or

Was it changed during transformation?
```

---

## 3. Use Reference Queries

Instead of repeatedly importing the same source, a staging query can reference the raw query.

Conceptually:

```text
Orders_Raw
    │
    ├──► Orders_Validation
    │
    └──► Orders_Final
```

This keeps source ingestion centralized and makes the transformation flow easier to understand.

---

## 4. Validate Row Counts

Important transformations should be checked for unexpected row loss.

Conceptually:

```text
Rows Before Transformation
            ↓
Transformation
            ↓
Rows After Transformation
            ↓
Difference
```

Operations that may change row counts include:

- filters
- duplicate removal
- joins
- grouping
- error removal
- null filtering

A changed row count is not automatically wrong.

It should simply be explainable.

---

## 5. Add an Index for Investigation

During troubleshooting, an index can make records easier to trace.

Example:

```powerquery
Table.AddIndexColumn(
    Source,
    "ValidationRowID",
    1,
    1,
    Int64.Type
)
```

The index is a technical investigation aid.

It should not be treated as a business key.

---

## 6. Set Data Types Explicitly

Do not rely unnecessarily on automatic type detection.

Example:

```powerquery
Table.TransformColumnTypes(
    Source,
    {
        {"OrderID", type text},
        {"PurchaseDate", type datetime},
        {"Revenue", type number},
        {"Quantity", Int64.Type}
    }
)
```

Explicit data types make transformations more predictable.

---

## 7. Business IDs Should Usually Remain Text

An identifier may contain only digits while still being a business identifier rather than a numeric measure.

Examples:

```text
Order ID
Invoice Number
Customer Number
External Reference
```

Converting these to numbers can:

- remove leading zeros
- change formatting
- prevent exact matching

Therefore, business identifiers should generally remain text unless there is a documented reason to convert them.

---

## 8. Trim Text Safely

Leading or trailing spaces can prevent exact matches.

Example:

```powerquery
Table.TransformColumns(
    Source,
    {
        {
            "OrderID",
            each if _ = null then null else Text.Trim(_),
            type text
        }
    }
)
```

This changes formatting without intentionally changing the business meaning of the identifier.

---

## 9. Clean Hidden Text Characters

Some imported text can contain non-printable characters.

A controlled cleaning step can use:

```powerquery
Table.TransformColumns(
    Source,
    {
        {
            "OrderID",
            each if _ = null then null else Text.Clean(_),
            type text
        }
    }
)
```

If both trimming and cleaning are needed:

```powerquery
Table.TransformColumns(
    Source,
    {
        {
            "OrderID",
            each
                if _ = null
                then null
                else Text.Trim(Text.Clean(_)),
            type text
        }
    }
)
```

Always preserve the raw source when transformations affect important business identifiers.

---

## 10. Standardize Categories Carefully

Categorical values may contain formatting differences such as:

```text
Amazon.de
amazon.de
AMAZON.DE
```

A standardized comparison field can be created without deleting the original value.

Example:

```powerquery
Table.AddColumn(
    Source,
    "Marketplace_Normalized",
    each
        if [Marketplace] = null
        then null
        else Text.Upper(Text.Trim([Marketplace])),
    type text
)
```

This produces a separate analytical field while preserving the original source value.

---

## 11. Preserve Original Values

Instead of overwriting:

```text
Marketplace
```

consider keeping:

```text
Marketplace_Raw
Marketplace_Normalized
```

The same principle can apply to:

```text
OrderID_Raw
OrderID_Normalized

Status_Raw
Status_Normalized
```

This improves traceability.

---

## 12. Detect Blank Business IDs

A validation flag can identify unusable identifiers.

Example:

```powerquery
Table.AddColumn(
    Source,
    "MissingOrderID",
    each
        [OrderID] = null
        or Text.Trim([OrderID]) = "",
    type logical
)
```

This is often preferable to immediately filtering those records out.

---

## 13. Create Validation Flags Instead of Deleting Records

Useful flags can include:

```text
MissingOrderID
UnknownMarketplace
UnknownStatus
InvalidDate
UnexpectedCurrency
DuplicateOrderID
AmountOutlier
```

The principle is:

```text
Detect
   ↓
Flag
   ↓
Quantify
   ↓
Investigate
   ↓
Decide
```

rather than:

```text
Detect
   ↓
Delete
```

---

## 14. Validate Status Values

If only certain statuses are expected, create a flag.

Example:

```powerquery
Table.AddColumn(
    Source,
    "StatusValid",
    each
        List.Contains(
            {"Shipped", "Canceled", "Pending"},
            [OrderStatus]
        ),
    type logical
)
```

Unexpected statuses can then remain visible for investigation.

The allowed status list should reflect the actual source system.

---

## 15. Validate Marketplace Values

Example:

```powerquery
Table.AddColumn(
    Source,
    "MarketplaceValid",
    each
        List.Contains(
            {
                "Amazon.de",
                "Amazon.fr",
                "Amazon.it",
                "Amazon.nl"
            },
            [Marketplace]
        ),
    type logical
)
```

This is only an example.

A production list should come from the actual expected marketplace population.

---

## 16. Validate Currency

A similar pattern can identify unexpected currencies.

```powerquery
Table.AddColumn(
    Source,
    "CurrencyValid",
    each
        List.Contains(
            {"EUR", "SEK", "PLN"},
            [Currency]
        ),
    type logical
)
```

Do not silently convert or aggregate currencies before the conversion logic has been defined.

---

## 17. Validate Dates

A simple validation flag can detect missing dates:

```powerquery
Table.AddColumn(
    Source,
    "PurchaseDateMissing",
    each [PurchaseDate] = null,
    type logical
)
```

Additional rules may check whether dates fall inside an expected range.

However, unusual dates should normally be flagged before they are removed.

---

## 18. Keep Date and DateTime Semantics Clear

If a source contains:

```text
2026-02-28 18:42:17
```

converting it to:

```text
2026-02-28
```

may be appropriate for daily reporting.

But the original timestamp can still be important for:

- process analysis
- cutoff rules
- timing investigations
- auditability

A useful approach can therefore be:

```text
PurchaseDateTime
PurchaseDate
```

rather than permanently replacing one with the other.

---

## 19. Extract a Date Without Losing the Original Timestamp

Example:

```powerquery
Table.AddColumn(
    Source,
    "PurchaseDateOnly",
    each Date.From([PurchaseDate]),
    type date
)
```

The original date-time column remains available.

---

## 20. Duplicate Detection

To inspect repeated identifiers:

```powerquery
Table.Group(
    Source,
    {"OrderID"},
    {
        {
            "RowCount",
            each Table.RowCount(_),
            Int64.Type
        }
    }
)
```

Filter the result to:

```text
RowCount > 1
```

to inspect repeated IDs.

---

## 21. Do Not Automatically Remove Duplicates

This transformation:

```powerquery
Table.Distinct(
    Source,
    {"OrderID"}
)
```

can be dangerous if repeated IDs represent:

- order lines
- split shipments
- settlement transactions
- refunds
- adjustments

Before using `Table.Distinct`, understand the table grain.

---

## 22. Composite Keys

Sometimes one field is not sufficient to identify a row.

A composite analytical key can be created:

```powerquery
Table.AddColumn(
    Source,
    "CompositeKey",
    each
        Text.Combine(
            {
                Text.From([OrderID]),
                Text.From([SKU])
            },
            "|"
        ),
    type text
)
```

This should only be used when the combined fields genuinely represent the required analytical grain.

---

## 23. Exact Matching First

When reconciling sources, begin with exact identifier matching.

Conceptually:

```text
Source A OrderID
        │
        ▼
Exact Match
        │
        ▼
Source B OrderID
```

Only after exact matching should alternative matching strategies be considered.

This prevents unnecessary normalization from creating false matches.

---

## 24. Left Anti Join for Unmatched Records

A useful reconciliation pattern is a left anti join.

Example:

```powerquery
Table.NestedJoin(
    SourceA,
    {"OrderID"},
    SourceB,
    {"OrderID"},
    "SourceB",
    JoinKind.LeftAnti
)
```

This returns Source A records that have no exact match in Source B.

It is useful for creating a residual investigation population.

---

## 25. Inner Join for Matched Records

Exact matches can be isolated with:

```powerquery
Table.NestedJoin(
    SourceA,
    {"OrderID"},
    SourceB,
    {"OrderID"},
    "SourceB",
    JoinKind.Inner
)
```

This allows matched records to be analyzed separately from residual records.

---

## 26. Full Outer Join for Reconciliation

A full outer join can support a complete reconciliation table:

```powerquery
Table.NestedJoin(
    SourceA,
    {"OrderID"},
    SourceB,
    {"OrderID"},
    "SourceB",
    JoinKind.FullOuter
)
```

After expansion, records can be classified as:

```text
Matched
Source A Only
Source B Only
```

The exact implementation depends on the source structure.

---

## 27. Create Match Status

After combining two sources, a status field can be added.

Conceptually:

```powerquery
Table.AddColumn(
    Reconciliation,
    "MatchStatus",
    each
        if [SourceAOrderID] <> null
            and [SourceBOrderID] <> null
        then "Matched"
        else if [SourceAOrderID] <> null
            and [SourceBOrderID] = null
        then "Source A Only"
        else if [SourceAOrderID] = null
            and [SourceBOrderID] <> null
        then "Source B Only"
        else "Unknown",
    type text
)
```

This is more informative than labeling every non-match as missing.

---

## 28. Analytical Identifier Normalization

During investigation, an additional normalized identifier may help test whether structured suffixes explain some unmatched records.

For example, if a validated analytical hypothesis concerns suffixes such as:

```text
ORDER-123_1
ORDER-123_2
```

a separate field may be created for analysis.

However:

> **Do not overwrite the original Order ID.**

The normalized value should remain a separate analytical field.

---

## 29. Example Controlled Suffix Investigation

A simple exploratory example might be:

```powerquery
Table.AddColumn(
    Source,
    "OrderID_Analytical",
    each
        if [OrderID] = null
        then null
        else
            let
                Parts = Text.Split([OrderID], "_"),
                LastPart = List.Last(Parts),
                IsNumericSuffix =
                    (try Number.FromText(LastPart) otherwise null) <> null
                    and List.Count(Parts) > 1
            in
                if IsNumericSuffix
                then Text.Combine(
                    List.RemoveLastN(Parts, 1),
                    "_"
                )
                else [OrderID],
    type text
)
```

This is an exploratory validation pattern.

It should not become production logic until the business meaning of the suffix has been confirmed.

---

## 30. Compare Original and Normalized Matching

A useful validation sequence is:

```text
Exact Matches
     ↓
Exact Unmatched
     ↓
Analytical Normalization
     ↓
Additional Matches
     ↓
Manual Validation
```

This allows the analyst to quantify whether normalization improves matching without silently changing the source data.

---

## 31. Amount Difference

After two sources have been aligned at the correct grain:

```powerquery
Table.AddColumn(
    Reconciliation,
    "AmountDifference",
    each
        [SourceAAmount] - [SourceBAmount],
    type number
)
```

This creates a record-level financial comparison.

---

## 32. Amount Match With Tolerance

A controlled tolerance can be applied:

```powerquery
Table.AddColumn(
    Reconciliation,
    "AmountMatch",
    each
        if [SourceAAmount] = null
            or [SourceBAmount] = null
        then "Not Comparable"
        else if
            Number.Abs(
                [SourceAAmount] - [SourceBAmount]
            ) <= 0.01
        then "Matched"
        else "Investigate",
    type text
)
```

The tolerance should be documented and justified.

---

## 33. Avoid Aggregate-Only Validation

Two sources may have the same total while containing different record-level values.

Therefore, Power Query validation should support both:

```text
Aggregate Comparison
        +
Record-Level Comparison
```

A zero total difference does not prove that every record matches.

---

## 34. Reason Codes

Residual records can be classified with evidence-based reason codes.

Examples:

```text
MATCHED
CANCELED
SETTLED_LATER
MARKETPLACE_DIFFERENCE
IDENTIFIER_VARIATION
AMOUNT_DIFFERENCE
SOURCE_A_ONLY
SOURCE_B_ONLY
UNRESOLVED
```

Reason codes improve:

- reporting
- auditability
- troubleshooting
- communication

Do not assign a reason code unless the available evidence supports it.

---

## 35. Keep Unresolved Records

Do not remove records simply because the cause cannot be explained.

A valid final reconciliation can contain:

```text
UNRESOLVED
```

These records represent known uncertainty.

Keeping them visible is more reliable than inventing an explanation.

---

## 36. Grouping for Validation

`Table.Group` can create useful control summaries.

Example:

```powerquery
Table.Group(
    Source,
    {"OrderStatus"},
    {
        {
            "Orders",
            each
                List.Count(
                    List.Distinct(
                        [OrderID]
                    )
                ),
            Int64.Type
        }
    }
)
```

Similar summaries can be created for:

```text
Marketplace
Currency
Month
Transaction Type
```

These distributions often reveal problems that grand totals hide.

---

## 37. Marketplace Distribution

A grouped validation table can reveal whether one source contains marketplaces that another source does not.

Conceptually:

```text
Marketplace   Orders
--------------------
Marketplace A   ...
Marketplace B   ...
Marketplace C   ...
```

This should be checked before interpreting an overall source difference.

---

## 38. Status Distribution

A status distribution can reveal:

- unexpected cancellations
- new status values
- blank statuses
- population differences

This is especially useful before defining a final order KPI.

---

## 39. Currency Distribution

Before aggregating international revenue:

```text
Currency   Records   Amount
---------------------------
EUR        ...
SEK        ...
PLN        ...
```

should be understood.

Do not sum different currencies into one financial KPI without an explicit conversion rule.

---

## 40. Remove Errors Carefully

Power Query provides operations for removing errors.

However, removing error rows can create silent data loss.

A better development process is:

```text
Detect Errors
     ↓
Count Errors
     ↓
Inspect Errors
     ↓
Understand Cause
     ↓
Decide Treatment
```

Only then should error records be removed or corrected.

---

## 41. Type Conversion Errors

A column conversion such as:

```powerquery
Table.TransformColumnTypes(
    Source,
    {{"Revenue", type number}}
)
```

may produce errors if the source contains:

- unexpected text
- wrong decimal separators
- corrupted values

Do not simply remove these errors without quantifying them.

---

## 42. Locale-Aware Conversion

International files may require explicit locale handling.

For example:

```powerquery
Table.TransformColumnTypes(
    Source,
    {
        {"Revenue", type number},
        {"PurchaseDate", type date}
    },
    "de-DE"
)
```

The correct locale depends on the actual source format.

Incorrect locale interpretation can materially change financial values and dates.

---

## 43. Detect Schema Changes

Production queries should be monitored for unexpected structural changes.

Potential changes include:

```text
Column Added
Column Removed
Column Renamed
Data Type Changed
```

A refresh that succeeds does not automatically prove that all expected business logic is still operating correctly.

---

## 44. Select Important Columns Explicitly

Where appropriate, explicitly selecting expected columns can make dependencies clearer.

Example:

```powerquery
Table.SelectColumns(
    Source,
    {
        "OrderID",
        "PurchaseDate",
        "OrderStatus",
        "Marketplace",
        "Currency",
        "Revenue"
    }
)
```

Whether missing columns should trigger an error or be tolerated depends on the production requirement.

---

## 45. API Pagination Validation

For API-based sources, verify that all pages are retrieved.

Conceptually:

```text
Request Page 1
      ↓
Next Token?
      ├── Yes → Request Next Page
      │
      └── No  → Complete
```

Potential problems include:

- page limits
- missing continuation tokens
- failed pages
- rate limits
- partial retries

A successful first request does not prove complete extraction.

---

## 46. Refresh Completeness

Useful controls can include:

```text
Expected Period
Actual Min Date
Actual Max Date
Row Count
Distinct Order Count
Last Refresh
```

This helps detect incomplete refreshes before they affect business KPIs.

---

## 47. Incremental Refresh Risks

Historical business records can change after their original creation.

Examples include:

- cancellation
- refund
- shipping update
- payment update
- financial adjustment

If only new records are refreshed, historical changes may remain outdated.

Refresh strategy should therefore reflect the business lifecycle.

---

## 48. Add Source Metadata

When multiple sources are combined, adding metadata can improve traceability.

For example:

```powerquery
Table.AddColumn(
    Source,
    "SourceSystem",
    each "MarketplaceReport",
    type text
)
```

Other useful metadata may include:

```text
SourceFile
ImportDate
ReportingPeriod
Marketplace
```

This makes later investigation easier.

---

## 49. Keep Transformation Names Clear

Instead of generic step names, prefer meaningful names such as:

```text
Changed Types
Trimmed Order IDs
Added Marketplace Validation
Filtered Reporting Period
Joined Settlement Data
Added Match Status
```

Clear transformation names make Power Query easier for another analyst to audit.

---

## 50. Document Non-Obvious Logic

When a transformation exists because of a specific business rule, document the reason.

For example:

```powerquery
// Analytical only:
// test whether terminal numeric suffixes explain unmatched settlement IDs.
// Original OrderID remains unchanged.
```

Comments should explain **why** a transformation exists.

---

## 51. Recommended Validation Query

A dedicated validation query can contain fields such as:

```text
OrderID
SourceSystem
Marketplace
OrderStatus
PurchaseDate
Currency
Amount
MissingOrderID
StatusValid
MarketplaceValid
CurrencyValid
DuplicateFlag
MatchStatus
ReasonCode
```

This creates a transparent diagnostic layer separate from the final management report.

---

## 52. Recommended Development Sequence

A reusable Power Query workflow is:

```text
1. Import source
        ↓
2. Preserve raw query
        ↓
3. Set explicit data types
        ↓
4. Validate row count
        ↓
5. Clean formatting
        ↓
6. Validate identifiers
        ↓
7. Validate dates
        ↓
8. Validate statuses
        ↓
9. Validate marketplace
        ↓
10. Validate currency
        ↓
11. Analyze duplicates
        ↓
12. Perform exact matching
        ↓
13. Investigate residual records
        ↓
14. Compare amounts
        ↓
15. Assign evidence-based reason codes
        ↓
16. Keep unresolved records visible
        ↓
17. Load validated model tables
```

---

## 53. Power Query Validation Checklist

Before loading transformed data into the model:

- [ ] Raw source preserved where practical
- [ ] Table grain documented
- [ ] Row counts checked
- [ ] Data types explicitly validated
- [ ] Business IDs stored appropriately
- [ ] Leading/trailing spaces reviewed
- [ ] Blank identifiers quantified
- [ ] Duplicate behavior understood
- [ ] Date semantics understood
- [ ] Marketplace values validated
- [ ] Status values validated
- [ ] Currency values validated
- [ ] Errors quantified before removal
- [ ] Joins checked for unexpected row multiplication
- [ ] Exact matching performed before normalization
- [ ] Identifier normalization kept traceable
- [ ] Amount tolerance documented
- [ ] Residual records retained
- [ ] Source metadata available where useful
- [ ] Important transformations documented
- [ ] Schema-change risk considered

---

## 54. What Not to Do

Avoid:

```text
Remove duplicates without understanding grain
```

Avoid:

```text
Delete unmatched records because they are inconvenient
```

Avoid:

```text
Replace unknown values silently
```

Avoid:

```text
Normalize identifiers destructively
```

Avoid:

```text
Remove errors without counting them
```

Avoid:

```text
Assume a successful refresh means complete data
```

Avoid:

```text
Transform data until two totals happen to match
```

---

## Final Principle

Power Query should not only prepare data for visualization.

It should help create a traceable path from:

```text
Raw Source
     ↓
Validated Source
     ↓
Standardized Data
     ↓
Reconciliation
     ↓
Semantic Model
     ↓
Power BI KPI
```

Every important transformation should answer two questions:

```text
Why is this transformation necessary?

What evidence shows that it preserves the intended business meaning?
```

The strongest transformation process is not the one that produces the cleanest-looking dataset.

It is the one that makes the path from raw data to analytical result understandable and reproducible.

> **Clean data is useful. Traceable data is trustworthy.**
