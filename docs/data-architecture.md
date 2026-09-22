# Data Architecture & Connections

A major challenge in this project was deciding how multiple data sources with different grains should interact inside Power BI.

The objective was not simply to connect every table that contained an Order ID.

The objective was to create a model in which:

- filters behave predictably
- KPIs use the intended business population
- financial values are not duplicated
- different reporting dates remain understandable
- reconciliation logic remains traceable
- source-specific differences remain visible

A central modeling principle was:

> **A technically possible relationship is not automatically a correct business relationship.**

---

## 1. Why the Architecture Matters

The project combined several fundamentally different types of data:

```text
Sellerboard
    → aggregated monthly KPI data

ERP / WaWi Orders
    → operational order data

ERP / WaWi Order Lines
    → detailed position and revenue data

Amazon Settlement
    → financial transaction data
```

Although these sources describe related business activity, they do not share the same grain.

Connecting them directly without understanding that difference can create:

- many-to-many relationships
- duplicate amounts
- ambiguous filter paths
- incorrect order counts
- incorrect revenue totals
- difficult-to-debug DAX
- misleading reconciliation results

The architecture therefore needed to preserve the meaning of each source before combining them analytically.

---

## 2. Recommended Analytical Architecture

A scalable reconciliation architecture can be structured as:

```text
                    SOURCE SYSTEMS
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
     Sellerboard     ERP / WaWi      Settlement
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                 POWER QUERY / STAGING
                         │
                         ▼
                    STANDARDIZATION
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
   SHARED DIMENSIONS             RECONCILIATION
   Date / Channel / etc.              LAYER
          │                             │
          └──────────────┬──────────────┘
                         ▼
                   SEMANTIC MODEL
                         │
                         ▼
                     DAX MEASURES
                         │
                         ▼
                    POWER BI REPORT
```

This separates several responsibilities that should not be mixed together.

---

## 3. Source Layer

The source layer should represent the original data as closely as practical.

Examples include:

```text
Sellerboard Export
ERP Order Data
ERP Order-Line Data
Amazon Settlement Export
```

At this stage, the primary objective is not to calculate final KPIs.

The objective is to understand:

- what the source contains
- what one row represents
- which fields identify a business object
- which dates are available
- which status fields exist
- which financial values are available

Keeping the source meaning clear makes later troubleshooting significantly easier.

---

## 4. Staging Layer

Each source should first be imported and cleaned independently.

Typical staging tasks include:

- assigning correct data types
- standardizing date fields
- trimming text
- checking blank values
- validating Order IDs
- checking duplicate records
- standardizing marketplace names
- checking decimal separators
- checking currencies
- documenting transformations

Conceptually:

```text
Raw Source
    ↓
Staging Query
    ↓
Validated / Standardized Data
```

The staging layer should create predictable input tables without changing their business meaning unnecessarily.

---

## 5. Preserve Raw Information

Transformations should remain traceable.

A useful pattern is:

```text
stg_AmazonOrders_Raw
        │
        ▼
stg_AmazonOrders_Clean
        │
        ▼
fact_AmazonOrders
```

The exact naming convention can vary.

The important idea is that cleaning logic should not silently destroy the ability to understand the original source.

This becomes particularly important when investigating:

- missing identifiers
- duplicate records
- unexpected status values
- marketplace differences
- amount differences

> **Clean data should remain traceable back to its source.**

---

## 6. Shared Date Dimension

A dedicated calendar table is an important part of a robust Power BI model.

Conceptually:

```text
             Date Dimension
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Orders     Revenue    Reporting
```

However, a calendar table does not automatically solve every date problem.

The project contained several different business dates:

- purchase date
- ERP creation date
- payment date
- shipping date
- refund date
- settlement transaction date

Each represents a different business event.

Therefore, the model must explicitly define which date controls each analysis.

---

## 7. Multiple Dates in the Same Business Process

A single order can move through several dates:

```text
Purchase
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

These events may occur on different days or even in different months.

For example:

```text
Order Analysis      → Purchase Date
ERP Operations      → Creation Date
Shipping Analysis   → Shipping Date
Settlement Analysis → Transaction Date
```

The correct date depends on the analytical question.

Using the wrong date can create an apparent source discrepancy even when the underlying records are correct.

---

## 8. Page-Specific Filter Context

One important issue discovered during this project was that different Power BI pages could receive their reporting period from different tables.

Conceptually:

```text
Management Page
      │
      ▼
Calendar[Date]
      │
      ▼
Management Measures
```

while another page could use:

```text
Amazon Analysis Page
      │
      ▼
MonthlyReport[Period]
      │
      ▼
Amazon Comparison Measures
```

A measure designed for one page may therefore return an unexpected result when reused on another page.

The arithmetic may still be correct.

The problem can be the source of the filter context.

This leads to an important rule:

> **Before changing a DAX calculation, verify which table and column are actually filtering the visual.**

---

## 9. Avoid Uncontrolled Fact-to-Fact Relationships

Direct relationships between detailed fact tables should be treated carefully.

For example:

```text
ERP Order Lines
      ↕
Amazon Settlement Transactions
```

can be dangerous because both sides may contain multiple rows for the same Order ID.

This can produce a many-to-many relationship.

Potential consequences include:

- duplicated revenue
- duplicated transaction amounts
- unexpected filtering
- ambiguous relationships
- incorrect totals
- difficult debugging

A relationship that Power BI allows technically may still be wrong analytically.

---

## 10. Prefer Clear Dimension-to-Fact Relationships

Where possible, a star-schema-style model is easier to understand and maintain.

Conceptually:

```text
                 DimDate
                    │
                    │
DimMarketplace ── FactOrders ── DimStatus
                    │
                    │
                DimProduct
```

For reconciliation projects, not every source will fit perfectly into one simple star schema.

However, the general principles remain useful:

- dimensions describe business entities
- fact tables contain measurable events
- relationships should have intentional filter direction
- business keys should be understood
- many-to-many relationships should not appear accidentally

---

## 11. Order Table vs. Order-Line Table

ERP systems often separate order headers and order positions.

Conceptually:

```text
Order
  │
  ├── Position 1
  ├── Position 2
  └── Position 3
```

This means:

```text
1 Order
≠
3 Order Lines
```

Order-count KPIs should therefore use the appropriate order-level business key.

Revenue calculations may instead require the order-line table.

This distinction is essential when order counts and revenue are used together in one dashboard.

---

## 12. Keep Order Population Consistent

If the order KPI excludes cancelled Amazon orders, the corresponding revenue KPI should not silently include those cancelled orders.

The intended population should be defined once and reused consistently.

Conceptually:

```text
Valid Amazon Orders
        │
        ├──────────────► Order Count
        │
        └──────────────► ERP Revenue
```

This prevents a dashboard from displaying:

```text
Order KPI    → Population A
Revenue KPI  → Population B
```

while both appear to describe the same business activity.

---

## 13. Controlled Filter Transfer with TREATAS

In this project, `TREATAS` was useful when a validated set of Amazon Order IDs needed to filter corresponding ERP records without introducing another physical relationship into the model.

A simplified pattern is:

```DAX
VAR ValidAmazonOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )

RETURN
CALCULATE(
    [ERP Net Revenue],
    TREATAS(
        ValidAmazonOrders,
        ERPOrders[ExternalOrderID]
    )
)
```

Conceptually:

```text
Validated Amazon Order IDs
            │
            ▼
         TREATAS
            │
            ▼
ERP External Order IDs
            │
            ▼
      ERP Net Revenue
```

This provides controlled filter transfer without requiring an additional physical relationship.

However, `TREATAS` should not be used blindly.

Before using it, verify:

- identifier compatibility
- identifier uniqueness
- table grain
- duplicate behavior
- intended filter direction
- order-status logic
- marketplace logic

---

## 14. Explicit Date Transfer

In some situations, the selected reporting period must also be transferred explicitly.

A simplified pattern is:

```DAX
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])

RETURN
CALCULATE(
    [Order Count],
    FILTER(
        ALL(AmazonOrders[PurchaseDate]),
        AmazonOrders[PurchaseDate] >= StartDate
            &&
        AmazonOrders[PurchaseDate] < EndDate + 1
    )
)
```

The pattern:

```text
< EndDate + 1
```

is useful when the source field contains a date-time value.

It allows the entire final selected day to be included without requiring the time component to be removed first.

---

## 15. Why ALL Requires Attention

Functions such as `ALL` can be necessary when rebuilding a filter explicitly.

However, they can also remove context that the report user expected to remain active.

Whenever `ALL` is used, ask:

- Which filter is being removed?
- Why is it being removed?
- Which filter is applied afterward?
- Could another slicer unintentionally be ignored?

Filter-removal logic should be deliberate and documented.

---

## 16. Reconciliation Layer

For larger projects, a dedicated reconciliation layer can simplify cross-source analysis.

Instead of repeatedly rebuilding matching logic inside individual measures, a reconciliation table can combine comparable business information.

Conceptually:

```text
OrderID
PurchaseDate
Marketplace
ERPStatus
MarketplaceStatus
ERP_Gross
ERP_Net
Marketplace_Gross
Marketplace_Net
VAT
SettlementFound
AmountDifference
MatchStatus
ReasonCode
```

The reconciliation layer answers:

> **For this business object, what does each system say?**

---

## 17. Example Match Status

A reconciliation layer can classify records using categories such as:

```text
MATCHED
CANCELLED
SETTLED_LATER
MARKETPLACE_MISMATCH
AMOUNT_DIFFERENCE
IDENTIFIER_VARIATION
ONLY_IN_ERP
ONLY_IN_MARKETPLACE
UNRESOLVED
```

This makes discrepancies measurable.

Instead of only reporting:

```text
Order Difference = 30
```

the analysis can potentially explain:

```text
Cancelled               18
Settled Later             6
Identifier Variation      3
Unresolved                3
```

The example values above are illustrative only.

---

## 18. Reason Codes

A separate reason code can make reconciliation results easier to audit.

For example:

```text
C01 = Cancelled order
D01 = Different reporting period
I01 = Identifier variation
M01 = Marketplace mismatch
R01 = Refund adjustment
T01 = Settlement timing
V01 = VAT difference
A01 = Amount difference
U01 = Unresolved
```

Reason codes make it possible to analyze not only the size of a discrepancy but also its cause.

---

## 19. Separate Operational Reporting from Reconciliation

A management dashboard and a reconciliation dashboard serve different purposes.

The management view should remain simple.

For example:

```text
Orders
Net Revenue
Open Orders
Shipping KPIs
```

A reconciliation view can provide deeper diagnostics:

```text
Source A Orders
Source B Orders
Order Difference
Matched Orders
Unmatched Orders
Cancelled Orders
Revenue Difference
Unresolved Records
```

This separation allows management reporting to remain readable while preserving analytical transparency.

---

## 20. Relationship Checklist

Before creating a Power BI relationship, verify:

- What is the grain of table A?
- What is the grain of table B?
- Which column is the proposed key?
- Is the key unique on either side?
- Can one business object appear multiple times?
- Is the relationship one-to-many or many-to-many?
- Which direction should filters propagate?
- Could another filter path already exist?
- Which date controls the analysis?
- Are marketplaces aligned?
- Are statuses aligned?
- Are currencies aligned?
- Are the financial values comparable?

If these questions cannot be answered, the relationship should not be added simply because matching columns exist.

---

## 21. Measure Debugging Checklist

When a KPI looks wrong, investigate the model before immediately rewriting the formula.

A useful sequence is:

```text
Unexpected KPI
      │
      ▼
Check Slicers
      │
      ▼
Check Page / Visual Filters
      │
      ▼
Identify Date Source
      │
      ▼
Check Relationships
      │
      ▼
Check Filter Direction
      │
      ▼
Check Data Grain
      │
      ▼
Check Business Population
      │
      ▼
Check DAX
      │
      ▼
Check Source Data
```

This reduces unnecessary trial-and-error changes.

---

## 22. Architecture Principles

The most important modeling principles from this project were:

### Preserve Source Meaning

Do not transform different source concepts into one definition before understanding them.

### Understand Grain

Know what one row represents before counting or summing.

### Control Relationships

Avoid relationships that introduce unintended many-to-many behavior.

### Make Dates Explicit

Different dates represent different business events.

### Keep Populations Consistent

Order and revenue KPIs should use compatible business populations.

### Transfer Filters Deliberately

Use relationships, explicit date logic, or `TREATAS` intentionally rather than relying on accidental filter propagation.

### Keep Reconciliation Traceable

A discrepancy should be explainable from source to transformation to model to measure.

---

## Final Architecture Principle

A robust Power BI model should create a traceable analytical path:

```text
Source
   ↓
Staging
   ↓
Standardization
   ↓
Business Definition
   ↓
Data Model
   ↓
Filter Context
   ↓
DAX Measure
   ↓
Validation
   ↓
Business Interpretation
```

If one of these stages is unclear, a KPI can be technically valid while still representing the wrong business result.

> **Good data architecture does not simply connect tables. It preserves business meaning while making analytical relationships explicit, controlled, and auditable.**
