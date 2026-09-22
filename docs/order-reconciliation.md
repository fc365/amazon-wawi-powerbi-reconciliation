# Order Reconciliation Methodology

Reconciling order counts across multiple systems requires more than comparing two totals.

Two systems may both display a KPI called **Orders** while using different:

- reporting dates
- order statuses
- marketplaces
- identifiers
- transaction grains
- refresh times
- business definitions

For that reason, the objective of this project was not to force the systems to show identical numbers.

The objective was to determine:

> **Which differences can be explained, which require investigation, and which remain unresolved?**

---

## 1. Reconciliation Workflow

The order reconciliation followed a structured process.

```text
Total Orders
     │
     ▼
Reporting Period
     │
     ▼
Business Definition
     │
     ▼
Order Status
     │
     ▼
Distinct Order IDs
     │
     ▼
Marketplace
     │
     ▼
Order ID Matching
     │
     ▼
Settlement Timing
     │
     ▼
Identifier Variations
     │
     ▼
Residual Differences
     │
     ▼
Manual Validation
```

Each step reduces the number of possible explanations before individual records are investigated.

---

## 2. Define the Reporting Period

The first step is to make sure that the datasets refer to the intended reporting period.

Several dates may exist for the same order:

- purchase date
- ERP creation date
- payment date
- shipping date
- refund date
- settlement transaction date

These dates describe different business events.

Conceptually:

```text
Purchase Date
     │
     ▼
ERP Creation
     │
     ▼
Payment
     │
     ▼
Shipping
     │
     ▼
Settlement
```

For operational Amazon order analysis, the purchase date may be the relevant date.

For settlement analysis, the settlement transaction date represents a different financial event.

Therefore:

> **Records should not be classified as missing simply because they appear in different reporting periods.**

---

## 3. Define What Counts as an Order

Before comparing totals, the business definition of an order must be explicit.

For example, an operational Amazon dataset may contain both:

```text
Shipped
Canceled
```

If the KPI is intended to represent valid operational orders, cancelled orders may need to be excluded.

A simplified DAX pattern is:

```DAX
Amazon Orders =
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    AmazonOrders[OrderStatus] <> "Canceled"
)
```

The important part is not the exact syntax.

The important part is the business rule:

```text
Which records belong to the KPI population?
```

Two systems can both display **Orders** while applying different status logic internally.

---

## 4. Validate the Raw Population First

Before applying exclusions, it is useful to understand the raw population.

Useful controls include:

```text
Total Rows
Distinct Order IDs
Blank Order IDs
Duplicate Order IDs
Orders by Status
Orders by Marketplace
```

This creates a baseline before records are removed.

A good reconciliation should be able to explain how the population changes from:

```text
Raw Population
      ↓
Validated Population
      ↓
Final KPI Population
```

---

## 5. Count Distinct Business Objects

A physical row should not automatically be treated as one order.

This is especially important when working with:

- order-line tables
- settlement transactions
- shipment tables
- payment transactions

For example:

```text
Order 001
    ├── Product A
    ├── Product B
    └── Product C
```

may represent:

```text
1 Order
3 Order Lines
```

Therefore:

> **Row count ≠ Order count**

When appropriate, a validated Order ID should be used as the business key.

A simplified control measure is:

```DAX
Distinct Orders =
DISTINCTCOUNT(Orders[OrderID])
```

---

## 6. Check for Blank Identifiers

Records without a usable business identifier can create hidden reconciliation problems.

Questions to check include:

- Is Order ID blank?
- Is the value null?
- Does the field contain only spaces?
- Is a placeholder value being used?
- Is the identifier malformed?

Conceptually:

```text
Valid ID
Blank ID
Malformed ID
Placeholder ID
```

Records without a reliable identifier should be quantified before matching begins.

They should not silently disappear from the analysis.

---

## 7. Check Duplicate Order IDs

A repeated Order ID is not automatically an error.

The meaning depends on the table grain.

For example:

```text
Order Table
OrderID should usually represent one business order
```

while:

```text
Order-Line Table
the same OrderID may legitimately appear several times
```

and:

```text
Settlement Table
the same OrderID may appear in several financial transactions
```

Before removing duplicates, determine why the identifier repeats.

> **Duplicate identifiers should be explained before they are deleted.**

---

## 8. Validate Order Status

Status logic can explain significant differences between systems.

Possible states may include:

- shipped
- cancelled
- pending
- partially processed
- refunded
- unknown

The reconciliation should quantify each status before deciding which statuses belong to the KPI.

A useful diagnostic view is:

```text
Status       Distinct Orders
----------------------------
Shipped             ...
Canceled            ...
Other               ...
```

This can reveal whether a difference is primarily caused by status definitions rather than missing records.

---

## 9. Validate the Marketplace

Orders should then be separated by marketplace or sales channel.

A major lesson from this project was:

> **Shipping country is not marketplace.**

For example:

```text
Sales Channel:     Amazon.de
Shipping Country:  France
```

and:

```text
Sales Channel:     Amazon.fr
Shipping Country:  Germany
```

can both be valid.

Therefore, marketplace reconciliation should use the dedicated marketplace or sales-channel field whenever available.

Useful validation fields include:

- sales channel
- marketplace
- shipping country
- billing country
- currency

These fields describe different concepts and should not be treated as interchangeable.

---

## 10. Compare Marketplace Distribution

Before matching individual orders, it can be useful to inspect the marketplace distribution.

Conceptually:

```text
Marketplace     Orders
----------------------
Amazon.de       ...
Amazon.fr       ...
Amazon.it       ...
Amazon.se       ...
Amazon.nl       ...
Amazon.pl       ...
```

This can reveal whether two datasets cover different marketplace populations.

A reconciliation should not compare:

```text
All Marketplaces
```

against:

```text
Amazon.de Only
```

and interpret the difference as missing orders.

---

## 11. Match Orders by Business Identifier

After period, status, and marketplace have been aligned, individual Order IDs can be compared.

A conceptual reconciliation table is:

```text
Order ID     ERP/WaWi     Settlement     Match Status
------------------------------------------------------
ORDER-001    Found        Found          Matched
ORDER-002    Found        Not Found      Investigate
ORDER-003    Not Found    Found          Investigate
```

An unmatched Order ID proves only:

> **The identifier was not found under the current matching rules and available data.**

It does not yet explain why.

---

## 12. Avoid Calling Unmatched Records "Missing" Too Early

Suppose:

```text
System A Orders = 1,000
System B Orders =   970
Difference       =    30
```

This proves that the systems differ by 30 under the current definitions.

It does **not** prove that 30 orders are missing.

Possible explanations include:

- cancellations
- different reporting periods
- marketplace differences
- settlement timing
- identifier variations
- duplicate logic
- source refresh timing
- different business definitions

During investigation, a safer description is:

> **Source difference**

The cause should only be assigned after evidence supports it.

---

## 13. Check Settlement Timing

Settlement data introduces an important timing issue.

An order purchased near the end of one month may generate financial transactions in a later settlement period.

Therefore:

```text
Purchase Month ≠ Settlement Month
```

A matching process limited to:

```text
Purchase Month = Settlement Month
```

can create false unmatched records.

A better workflow is:

```text
Order Purchased
      │
      ▼
Search Same Settlement Period
      │
      ▼
Search Later Settlement Periods
      │
      ▼
Still Not Found?
      │
      ▼
Residual Investigation
```

Only after the appropriate settlement horizon has been searched should the order be treated as a residual discrepancy.

---

## 14. Investigate Identifier Variations

Some systems may add suffixes or other variations to identifiers.

Conceptually:

```text
123-1234567-1234567
123-1234567-1234567_1
123-1234567-1234567_2
```

These variations may indicate:

- split transactions
- partial shipments
- adjustments
- partial processing
- source-specific transaction logic

For exploratory analysis, a normalized identifier can help discover relationships.

For example:

```text
123-1234567-1234567_1
            ↓
123-1234567-1234567
```

However:

> **Useful analytical normalization is not automatically a valid production rule.**

The suffix should only be removed permanently if its business meaning has been validated.

---

## 15. Exact Matching Before Fuzzy Logic

Order reconciliation should normally begin with exact identifier matching.

Conceptually:

```text
Exact Order ID Match
        ↓
Identifier Investigation
        ↓
Controlled Normalization
        ↓
Residual Cases
```

Fuzzy matching should not be the first solution for structured business identifiers.

A near match does not necessarily represent the same business object.

---

## 16. Reduce the Population Before Manual Investigation

Manual checking should happen only after automated rules have reduced the discrepancy population.

A useful reduction process is:

```text
All Records
    ↓
Correct Period
    ↓
Correct Status
    ↓
Correct Marketplace
    ↓
Distinct IDs
    ↓
Exact Matches Removed
    ↓
Later Settlement Matches Removed
    ↓
Identifier Variations Investigated
    ↓
Residual Cases
```

This transforms a potentially large manual task into a focused investigation.

---

## 17. Validate Residual Cases Individually

Remaining records should be checked against available operational information.

Useful fields include:

- Order ID
- purchase date
- ERP creation date
- order status
- cancellation timestamp
- payment status
- payment date
- shipping date
- marketplace
- shipping country
- gross amount
- net amount

This can distinguish cases such as:

```text
Cancelled Order
Settled Later
Marketplace Mismatch
Identifier Variation
Real Operational Order
Unresolved Difference
```

---

## 18. One Test Case Does Not Validate the Whole Dataset

A single successfully matched order is useful evidence.

It confirms that the logic works for that record.

It does not prove that every record follows the same pattern.

Therefore, conclusions should be stated precisely.

Instead of:

```text
The entire dataset is correct.
```

a stronger analytical statement is:

```text
This test case confirms the logic for this record.
```

A broader conclusion requires broader validation.

---

## 19. Use Evidence Levels

A useful way to avoid turning assumptions into facts is to classify findings by evidence strength.

### Confirmed

Directly supported by the relevant source data or source-system record.

### Empirically Supported

A pattern is supported by multiple observations but has not been formally confirmed by source-system documentation.

### Hypothesis

A possible explanation that still requires testing.

### Unresolved

The available evidence is insufficient to determine the cause.

This distinction is particularly important when investigating residual Order IDs.

---

## 20. Classify Reconciliation Results

A reconciliation process becomes easier to audit when each record receives a match classification.

Example:

```text
MATCHED
CANCELLED
SETTLED_LATER
MARKETPLACE_MISMATCH
IDENTIFIER_VARIATION
ONLY_IN_ERP
ONLY_IN_MARKETPLACE
UNRESOLVED
```

The exact categories should reflect the business process.

The objective is to turn one unexplained difference into measurable groups.

---

## 21. Add Reason Codes

Reason codes can provide a more structured explanation.

For example:

```text
C01 = Cancelled order
D01 = Different reporting period
I01 = Identifier variation
M01 = Marketplace mismatch
T01 = Settlement timing
U01 = Unresolved
```

A reconciliation table can then contain:

```text
OrderID | MatchStatus | ReasonCode
```

This improves:

- auditability
- troubleshooting
- reporting
- documentation
- future automation

---

## 22. Example Reconciliation Table

A simplified analytical structure could be:

```text
OrderID
PurchaseDate
Marketplace
ERPStatus
SettlementFound
SettlementPeriod
IdentifierNormalized
MatchStatus
ReasonCode
```

Additional financial fields can later be added for revenue reconciliation.

This provides one controlled place where the systems can be compared.

---

## 23. Separate Order Reconciliation from Revenue Reconciliation

An order can match by identifier while its financial amount still differs.

Therefore:

```text
Order Match
```

and:

```text
Amount Match
```

should be treated as separate validation questions.

Conceptually:

```text
Order ID Found?
      │
      ├── No  → Order Investigation
      │
      ▼
     Yes
      │
      ▼
Amount Comparable?
      │
      ├── No  → Financial Definition Investigation
      │
      ▼
     Yes
      │
      ▼
Amount Match?
```

This prevents order existence and financial correctness from being mixed into one test.

---

## 24. Build Small Control Measures

Large reconciliation measures should be supported by smaller diagnostic measures.

Examples:

```DAX
Raw Orders =
DISTINCTCOUNT(AmazonOrders[OrderID])
```

```DAX
Valid Orders =
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    AmazonOrders[OrderStatus] <> "Canceled"
)
```

```DAX
Canceled Orders =
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    AmazonOrders[OrderStatus] = "Canceled"
)
```

These controls answer one question at a time.

They help determine whether the problem is caused by:

- date context
- status
- marketplace
- identifier logic
- another transformation

---

## 25. Do Not Change Several Rules at Once

A reconciliation becomes difficult to debug when several transformations are changed simultaneously.

A stronger process is:

```text
Question
   │
   ▼
One Hypothesis
   │
   ▼
One Targeted Test
   │
   ▼
Evaluate Evidence
   │
   ▼
Next Question
```

This creates a traceable investigation.

It also prevents a correct result from being reached without knowing which change actually caused it.

---

## 26. Order Reconciliation Decision Tree

When order counts do not match, check in this order:

```text
Reporting Period
      ↓
Business Definition
      ↓
Order Status
      ↓
Distinct Order IDs
      ↓
Marketplace
      ↓
Identifier Matching
      ↓
Settlement Timing
      ↓
Identifier Variations
      ↓
Residual Cases
      ↓
Manual Validation
```

This sequence is useful because it starts with high-level population problems before moving into individual records.

---

## 27. What Not to Do

Several shortcuts can create misleading results.

### Do Not Assume Difference Means Missing Orders

A numerical difference proves only that the current populations differ.

### Do Not Count Transaction Rows as Orders

Understand the grain first.

### Do Not Use Shipping Country as Marketplace

Use the dedicated sales-channel or marketplace field.

### Do Not Compare Only the Same Settlement Month

Later settlement activity may be valid.

### Do Not Remove Identifier Suffixes Without Evidence

Normalization can hide meaningful distinctions.

### Do Not Validate the Entire Dataset from One Example

One example confirms one example.

### Do Not Force Equality

A correction factor should not be used simply because two systems are expected to match.

---

## 28. Keep Residual Differences Visible

Not every discrepancy can always be fully explained with the available data.

A professional reconciliation should not silently remove unresolved records.

Instead, they should remain visible as:

```text
UNRESOLVED
```

until additional evidence becomes available.

This is preferable to inventing a cause.

> **Unresolved is a valid analytical result.**

---

## 29. Recommended Order-Reconciliation KPIs

A dedicated analytical page can expose metrics such as:

```text
Source A Orders
Source B Orders
Order Difference
Matched Orders
Cancelled Orders
Settled-Later Orders
Identifier Variations
Marketplace Mismatches
Unresolved Orders
```

The operational dashboard can remain simple while the reconciliation page explains the differences.

---

## 30. Reusable Methodology

The complete reusable workflow is:

```text
1. Define the business question
              ↓
2. Define the reporting period
              ↓
3. Understand the source grain
              ↓
4. Identify the business key
              ↓
5. Count the raw population
              ↓
6. Define valid order statuses
              ↓
7. Validate marketplace
              ↓
8. Compare distinct Order IDs
              ↓
9. Perform exact matching
              ↓
10. Search later settlement periods
              ↓
11. Investigate identifier variations
              ↓
12. Reduce to residual cases
              ↓
13. Validate residuals individually
              ↓
14. Classify the evidence
              ↓
15. Document unresolved cases
```

---

## Final Takeaway

Order reconciliation is not primarily a counting problem.

It is a business-definition and data-quality problem.

A trustworthy result requires understanding:

```text
Population
    ↓
Period
    ↓
Status
    ↓
Grain
    ↓
Identifier
    ↓
Marketplace
    ↓
Timing
    ↓
Evidence
```

The goal is not to make two order totals identical at any cost.

The goal is to explain why they differ and to keep any remaining uncertainty visible.

> **A source difference is the beginning of an investigation, not the conclusion.**
