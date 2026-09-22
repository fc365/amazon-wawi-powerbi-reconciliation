# Data Quality Checklist

Reliable Power BI reporting depends on reliable source data.

A dashboard can contain technically correct DAX and still produce misleading results when the underlying data contains:

- missing records
- duplicate records
- invalid identifiers
- inconsistent statuses
- incorrect dates
- mixed currencies
- unexpected marketplace values
- incomplete refreshes
- schema changes

For that reason, data quality should be validated before reconciliation results are treated as trustworthy.

> **Data quality is not a final cleanup step. It is part of the analytical process.**

---

## 1. Data Quality Framework

A practical data-quality framework can evaluate the following dimensions:

```text
Completeness
     │
     ▼
Uniqueness
     │
     ▼
Consistency
     │
     ▼
Validity
     │
     ▼
Accuracy
     │
     ▼
Timeliness
     │
     ▼
Traceability
```

Each dimension answers a different question about the reliability of the data.

---

## 2. Completeness

Completeness asks whether the expected data is present.

Check:

- Are expected Order IDs available?
- Are important fields blank?
- Are all expected reporting periods available?
- Are all marketplaces represented?
- Are settlement files complete?
- Are required financial fields populated?
- Are expected source files available?
- Are recent records present?

A simple Power BI control can be:

```DAX
Orders Without ID =
CALCULATE(
    COUNTROWS(Orders),
    ISBLANK(Orders[OrderID])
)
```

A blank identifier should not silently disappear from reconciliation.

It should be quantified and investigated.

---

## 3. Missing Values

Not every blank value is an error.

For example:

```text
Shipping Date = blank
```

may be valid for an order that has not yet shipped.

However:

```text
Order ID = blank
```

may prevent reliable reconciliation.

Therefore, null analysis should consider the business meaning of each field.

A useful classification is:

```text
Expected Blank
Unexpected Blank
Conditionally Required
Always Required
```

---

## 4. Uniqueness

Uniqueness asks whether a business key appears as expected.

Before checking duplicates, determine the table grain.

For example:

```text
Order Header Table
OrderID expected once
```

while:

```text
Order-Line Table
OrderID can appear multiple times
```

and:

```text
Settlement Table
OrderID can appear in multiple financial transactions
```

Therefore:

> **Duplicate values are not automatically duplicate business records.**

---

## 5. Duplicate Analysis

When duplicate identifiers are found, investigate whether they represent:

- multiple order positions
- split shipments
- partial transactions
- refunds
- adjustments
- repeated imports
- genuine duplicate records

A useful duplicate analysis can contain:

```text
OrderID
RowCount
TransactionTypes
Dates
Amounts
Source
```

Do not remove duplicates until their meaning is understood.

---

## 6. Consistency

Consistency asks whether equivalent concepts are represented in a predictable way.

Examples of inconsistent values include:

```text
Amazon.de
Amazon DE
amazon.de
AMAZON.DE
```

or:

```text
Canceled
Cancelled
CANCELLED
```

Standardization can improve analysis, but the original source value should remain traceable.

---

## 7. Marketplace Consistency

Marketplace values should be reviewed separately from destination countries.

Validate:

- marketplace
- sales channel
- shipping country
- billing country
- currency

Do not assume these fields describe the same concept.

For example:

```text
Marketplace:      Amazon.de
Shipping Country: France
```

can be a valid combination.

---

## 8. Status Consistency

Status fields should be inspected for:

- expected values
- unexpected new values
- spelling differences
- blanks
- legacy values
- inconsistent capitalization

A simple diagnostic table can show:

```text
Status          Count
---------------------
Shipped         ...
Canceled        ...
Pending         ...
Unknown         ...
Blank           ...
```

New status values should be investigated before existing business logic is assumed to remain valid.

---

## 9. Validity

Validity asks whether values follow expected rules.

Examples include:

```text
Order ID follows expected structure
Date is a valid date
Currency is recognized
Marketplace is recognized
Amount is numeric
Status belongs to expected domain
```

Invalid values should normally be flagged rather than silently corrected.

---

## 10. Identifier Validation

Business identifiers are critical for reconciliation.

Check for:

- blanks
- leading spaces
- trailing spaces
- unexpected characters
- suffixes
- duplicate IDs
- truncated IDs
- inconsistent data types
- placeholder values

For example:

```text
ORDER-123
ORDER-123 
ORDER-123_1
```

may look similar but have different technical and potentially different business meanings.

---

## 11. Text Standardization

Useful Power Query operations may include:

- `Text.Trim`
- case standardization
- replacement of known formatting inconsistencies

However, normalization should be conservative.

For business identifiers, avoid transformations that could destroy meaningful information.

A safe principle is:

> **Standardize formatting before standardizing meaning.**

---

## 12. Date Validity

Date fields should be checked for:

- null dates
- impossible dates
- dates far outside the expected range
- incorrect data types
- time-zone effects
- date-time vs. date differences

Also verify the business meaning of each date.

For example:

```text
Purchase Date
Creation Date
Payment Date
Shipping Date
Settlement Date
```

are all valid dates but represent different events.

---

## 13. Date Sequence Checks

Some date relationships can be tested for plausibility.

Conceptually:

```text
Purchase Date
     ↓
ERP Creation
     ↓
Shipping
```

Unexpected sequences may require investigation.

However, automated rules should account for legitimate exceptions and source-system timing.

A suspicious sequence is evidence for investigation, not automatic proof of an error.

---

## 14. Accuracy

Accuracy asks whether a value correctly represents the real business event.

This is harder to prove automatically.

Validation may require comparison with:

- source-system records
- invoices
- payment records
- shipping records
- settlement transactions
- manually verified examples

A technically valid value is not automatically an accurate business value.

---

## 15. Financial Validity

Financial data should be checked for:

- unexpected negative values
- unexpected zero values
- unusually large values
- decimal precision
- gross/net consistency
- VAT consistency
- currency
- refunds
- discounts
- fees

The meaning of negative values should be understood before they are treated as errors.

For example, a negative amount may legitimately represent:

- VAT stored with negative sign
- refund
- fee
- discount
- adjustment

---

## 16. Gross / Net Plausibility

Where both gross and net values are available, they can be used for plausibility checks.

Conceptually:

```text
Gross
  │
  ├── VAT
  ▼
Net
```

However, do not assume one universal equation for every marketplace transaction.

Real-world complications can include:

- multiple VAT rates
- shipping VAT
- discounts
- refunds
- marketplace tax rules
- rounding

Use gross/net relationships as validation evidence rather than blind transformation rules.

---

## 17. Currency Validation

For international data, check:

- currency code
- marketplace
- amount
- reporting currency
- exchange-rate logic

Useful controls include:

```text
Currency   Record Count
-----------------------
EUR        ...
SEK        ...
PLN        ...
Other      ...
```

Unexpected currencies should be investigated before revenue is aggregated.

---

## 18. Timeliness

Timeliness asks whether data is available when expected.

Potential causes of temporary discrepancies include:

- delayed API synchronization
- scheduled source refresh
- delayed settlement processing
- failed refresh
- incomplete file delivery
- late source updates

Before classifying a discrepancy as a permanent data-quality issue, check whether the compared sources have the same data freshness.

---

## 19. Refresh Monitoring

A production-quality solution should ideally track:

```text
Source
Last Successful Refresh
Expected Refresh
Row Count
Status
```

For example:

```text
Amazon Orders     08:00     Expected 08:00     OK
ERP Orders        08:05     Expected 08:00     OK
Settlement        Previous Day         Expected  Daily
```

This helps distinguish analytical discrepancies from stale data.

---

## 20. Source Row-Count Monitoring

Row counts can provide an early warning for data-pipeline problems.

For example:

```text
Yesterday: 50,000 rows
Today:      5,000 rows
```

may indicate:

- failed import
- changed filter
- incomplete API pagination
- missing source files
- schema change

Row-count monitoring does not prove data correctness, but it can detect suspicious changes quickly.

---

## 21. API Pagination

When data is loaded from an API, a successful request does not necessarily mean that all records were downloaded.

Check:

- page size
- pagination token
- maximum result limit
- rate limits
- failed pages
- retry logic

A missing pagination loop can create silent data loss.

---

## 22. Incremental Refresh and Historical Changes

Incremental loading can create reconciliation problems when historical records change after their original load.

Examples include:

- later cancellations
- later refunds
- status changes
- delayed transactions
- corrections

If the refresh window is too short, historical changes may never be reloaded.

Therefore, incremental-refresh strategy should reflect the business process.

---

## 23. Schema Drift

Source systems can change over time.

Possible changes include:

- renamed columns
- removed columns
- new columns
- changed data types
- new status values
- new transaction types

A pipeline may even continue refreshing while business logic becomes incomplete.

Schema changes should therefore be monitored.

---

## 24. Data-Type Validation

Incorrect data types can cause subtle errors.

Examples:

```text
Order ID stored as number instead of text
Date stored as text
Revenue stored as text
Decimal separator interpreted incorrectly
```

Business identifiers should generally not be converted to numeric values simply because they contain digits.

Leading zeros or formatting can be meaningful.

---

## 25. Decimal Separator and Locale

International data can use different conventions.

For example:

```text
1,234.56
```

and:

```text
1.234,56
```

represent the same conceptual value under different locale conventions.

Incorrect locale settings can transform financial data incorrectly.

Power Query import settings should therefore be validated for the source format.

---

## 26. Rounding Validation

Small differences may result from rounding at different levels.

Compare:

```text
Round each transaction
        ↓
Sum
```

with:

```text
Sum precise transactions
        ↓
Round total
```

The result may differ slightly.

Document the expected rounding rule before introducing a tolerance.

---

## 27. Outlier Detection

Unusually large or small values can reveal data-quality problems.

Potential controls include:

- largest orders
- negative revenue
- zero-value orders
- extreme quantities
- unusual VAT ratios
- unusually old dates

An outlier is not automatically wrong.

It is a record that deserves investigation.

---

## 28. Distribution Checks

Data-quality analysis should not rely only on totals.

Useful distributions include:

```text
Orders by Status
Orders by Marketplace
Orders by Currency
Orders by Month
Revenue by Marketplace
Revenue by Status
```

A sudden distribution change can reveal a problem that aggregate totals hide.

---

## 29. Referential Integrity

When multiple tables are related, check whether business keys exist on both sides.

For example:

```text
Order Header
     │
     ▼
Order Lines
```

Questions include:

- Are there order lines without an order header?
- Are there order headers without positions?
- Are settlement Order IDs absent from the operational order source?
- Are foreign keys blank?

Unmatched keys should be quantified.

---

## 30. Relationship Cardinality

Before creating a relationship, verify whether the proposed key is unique.

A supposed:

```text
One-to-Many
```

relationship can become:

```text
Many-to-Many
```

if duplicates exist on the expected dimension side.

Cardinality should therefore be validated from the data, not assumed from the column name.

---

## 31. Power Query Validation Strategy

Power Query can be used not only for transformation but also for transparent validation.

A robust flow is:

```text
Raw Source
    ↓
Data Types
    ↓
Text Standardization
    ↓
Identifier Validation
    ↓
Date Validation
    ↓
Marketplace Validation
    ↓
Duplicate Check
    ↓
Business Rule Validation
    ↓
Clean Staging Table
```

Each transformation should have a documented purpose.

---

## 32. Avoid Silent Data Loss

Operations such as:

- removing errors
- removing duplicates
- filtering rows
- replacing null values
- changing data types

can remove or alter important records.

Before and after important transformations, compare:

```text
Row Count Before
Row Count After
Difference
```

Unexpected changes should be investigated.

---

## 33. Preserve the Raw Source

Whenever practical, preserve a raw or minimally transformed source query.

Conceptually:

```text
Raw
 ↓
Clean
 ↓
Model
```

This provides a reference when a later transformation produces an unexpected result.

Without the raw layer, it can be difficult to determine whether a problem originated:

- in the source
- in Power Query
- in the model
- in DAX

---

## 34. Data Quality Flags

Instead of deleting suspicious records, create validation flags where appropriate.

Examples:

```text
MissingOrderID
DuplicateOrderID
UnknownMarketplace
UnknownStatus
InvalidDate
UnexpectedCurrency
AmountOutlier
```

This allows problematic records to remain visible while still supporting analysis.

---

## 35. Severity Levels

Data-quality findings can also be classified by severity.

For example:

```text
INFO
WARNING
CRITICAL
```

Possible interpretation:

```text
INFO
Known and acceptable source behavior

WARNING
Requires review but does not necessarily invalidate reporting

CRITICAL
Can materially distort a KPI or prevent reliable reconciliation
```

This helps prioritize investigation.

---

## 36. Evidence Levels

Data-quality conclusions should reflect the available evidence.

### Confirmed

Directly supported by source data or source-system validation.

### Empirically Supported

Repeated evidence strongly supports the interpretation.

### Hypothesis

Possible explanation requiring further testing.

### Unresolved

Available data is insufficient for a reliable conclusion.

This prevents uncertain assumptions from becoming permanent business rules.

---

## 37. Pre-Reconciliation Checklist

Before comparing two systems, confirm:

### Source

- [ ] Correct source selected
- [ ] Expected files or tables available
- [ ] Refresh completed
- [ ] Expected reporting period available

### Grain

- [ ] Grain of each table documented
- [ ] Order-level vs. order-line level understood
- [ ] Transaction-level tables identified

### Identifier

- [ ] Business key identified
- [ ] Blank identifiers quantified
- [ ] Duplicate behavior understood
- [ ] Identifier formatting checked

### Date

- [ ] Relevant business date defined
- [ ] Date type validated
- [ ] Date-time behavior understood
- [ ] Time-zone risk considered

### Status

- [ ] Expected status values identified
- [ ] Cancelled records understood
- [ ] Unknown statuses reviewed

### Marketplace

- [ ] Marketplace field identified
- [ ] Shipping country not confused with marketplace
- [ ] Marketplace scope aligned

### Financial

- [ ] Gross/net definition documented
- [ ] VAT treatment understood
- [ ] Currency validated
- [ ] Refund treatment understood
- [ ] Fees separated from revenue where required

---

## 38. Power BI Model Checklist

Before trusting the report:

- [ ] Relationship cardinality validated
- [ ] Filter direction intentional
- [ ] No accidental many-to-many relationships
- [ ] Date relationships understood
- [ ] Inactive relationships documented
- [ ] Slicer source verified
- [ ] Cross-table filters validated
- [ ] `TREATAS` keys validated
- [ ] `ALL` usage reviewed
- [ ] Control measures tested

---

## 39. Final KPI Validation Checklist

Before calling a KPI validated:

- [ ] Business definition documented
- [ ] Source documented
- [ ] Population validated
- [ ] Reporting period validated
- [ ] Marketplace scope validated
- [ ] Status logic validated
- [ ] Currency validated
- [ ] Gross/net definition validated
- [ ] VAT treatment validated
- [ ] Record-level samples checked
- [ ] Aggregate total checked
- [ ] Remaining difference quantified
- [ ] Unresolved cases documented

A KPI should not be considered correct simply because the number looks plausible.

---

## 40. Data Quality Monitoring for Production

For a production environment, useful automated checks could include:

```text
Source Refresh Status
Row Count Change
Blank Order IDs
Duplicate Order IDs
Unknown Status Values
Unknown Marketplaces
Unexpected Currencies
Revenue Outliers
Unmatched Business Keys
Reconciliation Difference
```

Alerts can then be triggered when values exceed agreed thresholds.

---

## 41. Recommended Data Quality Table

A reusable monitoring table could contain:

```text
CheckID
CheckName
Source
CheckDate
ExpectedValue
ActualValue
Difference
Severity
Status
Comment
```

Example statuses:

```text
PASS
WARNING
FAIL
UNRESOLVED
```

This turns data-quality checks into an auditable process.

---

## 42. Data Quality Is Continuous

Data quality should not be treated as a one-time project phase.

Source systems change.

Business processes change.

New marketplaces appear.

New statuses appear.

Financial rules change.

APIs change.

Therefore:

```text
Validate
   ↓
Monitor
   ↓
Investigate
   ↓
Document
   ↓
Improve
   ↓
Validate Again
```

is a continuous cycle.

---

## Final Data Quality Principle

A reliable analytical system requires more than correct formulas.

It requires confidence in:

```text
Completeness
     ↓
Uniqueness
     ↓
Consistency
     ↓
Validity
     ↓
Accuracy
     ↓
Timeliness
     ↓
Traceability
```

The objective is not to create data that merely looks clean.

The objective is to understand which data can be trusted, which data requires investigation, and which uncertainty must remain visible.

> **Good data quality does not mean that every record is perfect. It means that the quality, limitations, and remaining uncertainty of the data are known and transparent.**
