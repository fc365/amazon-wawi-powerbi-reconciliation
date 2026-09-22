# Amazon × ERP/WaWi Data Reconciliation in Power BI

### A Real-World Data Quality & Reconciliation Case Study

This project documents a real-world Power BI data reconciliation challenge involving multiple e-commerce and ERP data sources.

The goal was not simply to build a dashboard, but to understand why orders and revenue reported by different systems did not always match.

The project focuses on the reconciliation of:

- Amazon marketplace data
- Sellerboard reporting data
- ERP/WaWi order data
- Amazon settlement data
- Power BI reporting and DAX calculations

During the analysis, several common real-world data problems appeared: different order definitions, cancellations, settlement delays, marketplace differences, gross vs. net revenue, VAT, split transactions, inconsistent date contexts, and Power BI filter-context issues.

Rather than forcing the systems to produce identical numbers, the objective became to identify, explain, quantify, and transparently report the remaining differences.

> **Note:** All company-sensitive information, customer information, order IDs, internal system details, and financial values used in the public version of this project are anonymized or replaced with synthetic examples.

---

## The Business Problem

At first glance, the task seemed simple: compare Amazon-related orders and revenue across different systems and display the results in Power BI.

In practice, the same business activity was represented differently across the available data sources.

An order could:

- exist in the ERP/WaWi but appear in the Amazon settlement at a later date
- be cancelled in one system while still being present in another dataset
- belong to a different marketplace than expected
- contain split or suffixed transaction identifiers
- have different purchase, creation, shipping, payment, and settlement dates
- be represented as gross revenue in one source and net revenue in another
- include VAT, refunds, fees, discounts, or shipping components differently

This created an important analytical problem:

> **A numerical difference between two systems does not automatically mean that data is missing or incorrect.**

Before correcting any KPI, the underlying business definition, data grain, date logic, order status, marketplace, and revenue definition must first be understood.

### Core Questions

The reconciliation therefore focused on questions such as:

1. What exactly counts as an order in each system?
2. Which date determines the reporting period?
3. How should cancelled orders be handled?
4. Are Order IDs truly comparable across the systems?
5. Which marketplace does an order belong to?
6. Are the compared revenue values gross, net, settlement, or payout values?
7. How are VAT, refunds, fees, discounts, and shipping treated?
8. Can settlement transactions occur in a later month than the original purchase?
9. Is a Power BI difference caused by the source data, the data model, or the DAX filter context?
10. When is a remaining difference a genuine data-quality issue rather than a definitional difference?

These questions became the foundation of the reconciliation methodology used throughout this project.

---

## Data Sources & Data Grain

One of the most important lessons from this project was that data sources should not be compared only because they describe the same business process.

The systems used in this analysis represent Amazon activity at different levels of detail and at different points in time.

### 1. Sellerboard

Sellerboard was used as a monthly reporting source for Amazon performance.

The available export contained aggregated metrics such as:

- Orders
- Organic Sales
- PPC Sales
- VAT
- Refunds
- Amazon Fees
- Shipping Costs
- Estimated Payout
- Net Profit

**Data grain:** aggregated reporting period / month.

This means that the Sellerboard export could be used for KPI comparison, but not for a direct order-by-order reconciliation because individual Order IDs were not available in the monthly dataset.

---

### 2. ERP / WaWi

The ERP/WaWi system contained detailed operational order information.

Relevant fields included information such as:

- External Order ID
- Purchase Date
- Order Creation Date
- Order Status
- Cancellation Status
- Sales Channel
- Shipping Country
- Payment Information
- Order Positions
- Gross and Net Values

**Data grain:** order and order-line level.

This source allowed individual Amazon orders to be investigated and matched against other transaction-level datasets.

A key lesson was that the shipping country should not automatically be interpreted as the Amazon marketplace. The sales-channel field was required to distinguish marketplaces such as Amazon.de, Amazon.fr, Amazon.it, and others.

---

### 3. Amazon Settlement Data

Amazon settlement files contained financial transactions related to marketplace activity.

Typical fields included:

- Transaction Date
- Settlement ID
- Transaction Type
- Amazon Order ID
- SKU
- Quantity
- Marketplace
- Product Sales
- Product VAT
- Shipping Credits
- Promotional Discounts
- Marketplace Withheld Tax
- Selling Fees
- Fulfilment Fees
- Other Transaction Fees
- Transaction Total

**Data grain:** financial transaction / settlement-line level.

This is fundamentally different from an ERP order table.

One customer order may generate multiple financial transactions, and those transactions may be recorded in a later reporting period than the original purchase.

Therefore:

> **Purchase month ≠ Settlement month**

This became one of the most important rules of the reconciliation.

---

### Why Data Grain Matters

A direct comparison such as:

`Sellerboard Orders = ERP Rows = Settlement Rows`

would be analytically incorrect.

Before comparing KPIs, each source must first be understood in terms of:

- business definition
- granularity
- unique identifier
- reporting date
- status logic
- marketplace
- currency
- gross/net treatment
- tax treatment

Only after these definitions are aligned should differences be interpreted as potential data-quality problems.

This principle prevented several apparent discrepancies from being incorrectly classified as missing orders or missing revenue.

---

## Data Architecture & Connections

A major challenge in this project was deciding how the different sources should interact inside Power BI.

The most important lesson was:

> **Data sources should not be connected directly just because they contain information about the same orders.**

Sellerboard, ERP/WaWi, and Amazon Settlement data have different grains and different business meanings. Creating relationships without considering this can introduce duplicate rows, incorrect totals, ambiguous filter paths, or many-to-many relationships.

### Recommended Architecture

A more robust analytical architecture separates source data, transformation logic, shared dimensions, reconciliation logic, and reporting measures.

```text
Sellerboard ───────┐
                   │
ERP / WaWi ────────┼──> Power Query / Staging
                   │           │
Settlement ────────┘           ▼
                         Standardization
                               │
                   ┌───────────┴───────────┐
                   ▼                       ▼
              Date Dimension        Reconciliation Layer
                   │                       │
                   └───────────┬───────────┘
                               ▼
                        Semantic Model
                               │
                               ▼
                         DAX Measures
                               │
                               ▼
                       Power BI Report
```

### 1. Staging Layer

Each source should first be imported and cleaned independently.

Typical staging tasks include:

- assigning correct data types
- standardizing dates
- trimming text values
- checking null values
- validating Order IDs
- standardizing marketplace names
- checking decimal separators and currencies
- detecting duplicate records
- documenting source-specific transformations

The purpose of this layer is not to calculate final KPIs. It is to create predictable and traceable input tables.

### 2. Shared Date Dimension

A dedicated calendar table should be used whenever possible to provide consistent report filtering.

However, one calendar selection does not mean that every source uses the same business date.

For example:

- ERP reporting may use order creation date
- Amazon operational analysis may use purchase date
- shipping analysis may use shipping date
- settlement analysis may use transaction date

This distinction became critical in this project.

A measure that worked correctly on one report page produced incorrect results on another page because the pages were controlled by different date contexts.

The solution was not to change the arithmetic. The correct date filter had to be transferred explicitly to the relevant source.

### 3. Avoid Uncontrolled Fact-to-Fact Relationships

Direct relationships between detailed ERP order lines and Amazon settlement transactions can be dangerous.

Both tables may contain multiple rows for the same Order ID.

A direct relationship can therefore create:

- many-to-many relationships
- duplicated amounts
- incorrect aggregations
- ambiguous filtering
- difficult-to-debug results

For reconciliation tasks, it can be safer to create a dedicated reconciliation layer or use controlled filter transfer in measures.

### 4. Controlled Filter Transfer with DAX

In this project, `TREATAS` was useful when a validated set of Amazon Order IDs needed to filter ERP/WaWi orders without introducing another physical relationship into the model.

A simplified pattern is:

```DAX
VAR ValidAmazonOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )

RETURN
CALCULATE(
    [Net Revenue],
    TREATAS(
        ValidAmazonOrders,
        ERPOrders[ExternalOrderID]
    )
)
```

This pattern should not be copied blindly. The identifier, grain, filter direction, and business definition must first be validated.

### 5. Connection Checklist

Before creating a relationship or cross-source measure, verify:

- What is the grain of both tables?
- Is the proposed key unique on either side?
- Can one order appear multiple times?
- Are Order IDs formatted consistently?
- Which date should control the analysis?
- Does the relationship create a many-to-many path?
- Is the filter direction intentional?
- Are cancelled records included?
- Are marketplaces aligned?
- Are currencies aligned?
- Are the compared values gross or net?
- Could a transaction appear in a later accounting period?

> **A technically valid Power BI relationship is not automatically a correct business relationship.**

---

## Order Reconciliation Methodology

Comparing order totals across multiple systems requires more than placing two KPIs next to each other.

In this project, the reconciliation was performed step by step. Each step reduced the number of possible causes before individual discrepancies were investigated.

The general workflow was:

```text
Total Orders
     │
     ▼
Reporting Period
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

### 1. Define the Reporting Period

The first step was to make sure that the compared datasets referred to the intended reporting period.

This sounds simple, but several different dates existed across the systems:

- purchase date
- ERP creation date
- payment date
- shipping date
- settlement transaction date

These dates describe different business events.

For operational Amazon order analysis, the purchase date was the relevant date. For settlement analysis, however, the settlement transaction date represented a different process.

Therefore, records were not classified as missing simply because they appeared outside the original purchase month.

---

### 2. Define What Counts as an Order

The next step was to define the order population before comparing totals.

For example, the operational Amazon dataset contained both shipped and cancelled orders.

A simplified Power BI pattern was:

```DAX
Amazon Orders =
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    AmazonOrders[OrderStatus] <> "Canceled"
)
```

The important point is not the exact DAX syntax.

The important point is that the business definition must be explicit.

Two systems can both display a metric called **Orders** while applying different status rules internally.

---

### 3. Count Distinct Order IDs

Row counts were not automatically treated as order counts.

Before comparing systems, the analysis checked:

- total rows
- distinct Order IDs
- duplicate Order IDs
- blank identifiers
- repeated identifiers
- transaction-level duplicates

This was particularly important for settlement data because one customer order can generate multiple financial transaction rows.

Therefore:

> **Row count ≠ Order count**

A `DISTINCTCOUNT` of a validated business identifier is often more meaningful than counting physical rows.

---

### 4. Validate the Marketplace

Orders were then separated by marketplace.

One important finding was that the shipping country could not safely be used as the marketplace identifier.

For example, an order shipped to Germany does not automatically prove that the order originated from the German Amazon marketplace.

The dedicated sales-channel or marketplace field was therefore used instead.

A marketplace validation should check fields such as:

- sales channel
- marketplace
- shipping country
- billing country
- currency

These fields describe different concepts and should not be treated as interchangeable.

---

### 5. Match Orders by Business Identifier

After period, status, and marketplace were aligned, individual Order IDs could be compared.

A reconciliation table can conceptually classify records as:

```text
Order ID        ERP/WaWi        Settlement        Match Status
---------------------------------------------------------------
ORDER-001       Found           Found             Matched
ORDER-002       Found           Not Found         Investigate
ORDER-003       Not Found       Found             Investigate
```

The purpose of this step is not immediately to label unmatched records as errors.

An unmatched Order ID only means:

> **The identifier was not found under the current matching rules and available data.**

The reason still needs to be investigated.

---

### 6. Check Settlement Timing

A major source of apparent differences was timing.

An order purchased near the end of one month may generate settlement transactions in the following month.

Therefore, matching only:

```text
Purchase Month = Settlement Month
```

can incorrectly classify valid orders as missing.

The analysis therefore searched settlement data beyond the original purchase month before classifying an order as a residual discrepancy.

This leads to an important reconciliation rule:

> **Always distinguish event date from accounting or settlement date.**

---

### 7. Investigate Identifier Variations

Some transaction identifiers contained additional suffixes or variations.

A simplified example:

```text
123-1234567-1234567
123-1234567-1234567_1
123-1234567-1234567_2
```

These patterns may indicate split transactions, partial processes, or source-specific identifier logic.

For analysis, normalized identifiers can help discover relationships between records.

However, normalization should not automatically become production logic.

Before removing suffixes permanently, it should be confirmed that the suffix does not represent a meaningful business distinction.

> **Useful analytical normalization is not automatically a valid production rule.**

---

### 8. Reduce the Problem Before Manual Investigation

Manual investigation should happen only after automated checks have reduced the discrepancy population.

Instead of manually reviewing thousands of orders, the process progressively reduced the dataset using:

- date validation
- status filtering
- distinct Order IDs
- marketplace filtering
- exact ID matching
- settlement-period expansion
- identifier analysis

Only the remaining unexplained records were investigated individually.

This makes reconciliation more efficient and creates a reproducible analytical process.

---

### 9. Validate Residual Cases Individually

Remaining records were checked against available operational information.

Useful validation fields can include:

- Order ID
- order status
- cancellation timestamp
- payment status
- purchase date
- creation date
- shipping date
- marketplace
- shipping country
- gross value
- net value

This step can distinguish between cases such as:

- cancelled orders
- delayed settlement transactions
- marketplace mismatches
- identifier differences
- genuinely unresolved records

A single validated order should only be treated as evidence for that specific case unless the same pattern is demonstrated across the wider population.

> **One matching example does not prove that the entire dataset is correct.**

---

### 10. Keep Unresolved Differences Visible

Not every discrepancy can always be fully explained with the available data.

In this project, a small residual population remained after the systematic reconciliation steps.

Those records were not silently removed and no arbitrary correction factor was applied.

They were documented as unresolved differences requiring further source-system or business validation.

This is an important data-quality principle:

> **An unexplained difference should remain visible until there is evidence explaining it.**

The objective of reconciliation is not to force two systems to match.

The objective is to determine which differences are:

- explained by business definitions
- explained by timing
- explained by status
- explained by marketplace
- explained by identifier structure
- caused by transformation or model logic
- still unresolved

---

## Revenue Reconciliation Methodology

Reconciling revenue across e-commerce systems is more complex than comparing two totals.

Different systems can report different financial values for the same business activity while all calculations are technically correct.

The first question should therefore not be:

> **Why are the numbers different?**

The first question should be:

> **Are we actually comparing the same financial concept?**

### 1. Define the Revenue Metric

Several financial metrics can exist for the same order:

- gross sales
- net sales
- VAT
- shipping revenue
- promotional discounts
- refunds
- marketplace fees
- fulfilment fees
- settlement amount
- estimated payout
- profit

These values should not be treated as interchangeable.

A useful conceptual distinction is:

```text
Gross Revenue
      │
      ├── VAT
      ▼
Net Revenue
      │
      ├── Refunds / Adjustments
      ├── Marketplace Fees
      ├── Fulfilment Fees
      └── Other Transactions
      ▼
Settlement / Payout
```

The exact calculation depends on the source system and accounting definition.

Therefore:

> **Revenue ≠ Settlement ≠ Payout ≠ Profit**

---

### 2. Understand the Source Definition

Before building a Power BI measure, each source field should be understood independently.

For example, a monthly reporting source may contain separate values for:

- organic sales
- advertising-attributed sales
- VAT
- refunds
- fees
- shipping costs
- estimated payout

A simplified reporting measure may combine sales components:

```DAX
Gross Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
```

If VAT is stored separately as a negative value, a net-like analytical measure may use:

```DAX
Net Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
    + SUM(MonthlyReport[VAT])
```

This formula is source-specific.

It should only be used after confirming how VAT and the sales fields are represented in the actual export.

> **Never copy a revenue formula without validating the source semantics first.**

---

### 3. Compare Equivalent Financial Concepts

One of the easiest reconciliation mistakes is comparing values that look similar but represent different concepts.

Examples of invalid comparisons include:

```text
ERP Net Revenue      vs. Marketplace Payout
Gross Order Value    vs. Net Revenue
Sales Revenue        vs. Settlement Total
Purchase-Month Sales vs. Settlement-Month Transactions
```

A better comparison is:

```text
Source A Net Revenue
        vs.
Source B Net Revenue
```

with aligned:

- order population
- marketplace
- reporting period
- cancellation logic
- tax treatment
- currency
- revenue components

Only then does the remaining difference become analytically meaningful.

---

### 4. Validate Gross and Net Values

When gross and net values are available, VAT can be used as an important plausibility check.

Conceptually:

```text
Gross Revenue = Net Revenue + VAT
```

However, real marketplace data can contain additional tax logic, discounts, refunds, shipping components, or marketplace-specific tax treatment.

Therefore, the equation should be treated as a validation concept rather than blindly assumed for every transaction.

At order level, comparing gross and net values can help identify whether two systems are storing:

- the same customer-facing amount
- tax-inclusive amounts
- tax-exclusive amounts
- settlement-adjusted amounts

---

### 5. Separate Revenue from Marketplace Fees

Marketplace fees should not automatically be deducted when the objective is to calculate sales revenue.

Settlement datasets can contain:

- selling fees
- fulfilment fees
- transaction fees
- shipping-related charges
- other adjustments

These values explain how money moves between the marketplace and the seller.

They do not automatically redefine the original sales revenue.

This distinction is essential when comparing ERP revenue with marketplace financial data.

---

### 6. Treat Refunds Separately

Refunds can create another timing problem.

An order may be purchased in one period and refunded in a later period.

Therefore:

```text
Purchase Period ≠ Refund Period
```

A monthly sales comparison can become misleading if one source reports revenue by purchase date while another includes later financial adjustments.

Refund analysis should therefore consider:

- original Order ID
- original purchase date
- refund transaction date
- refunded amount
- tax adjustment
- reporting-period definition

---

### 7. Use the Same Order Population

Revenue comparisons are only meaningful when the compared systems refer to the same intended population.

For example, if cancelled orders are excluded from the operational Amazon order KPI, the related ERP revenue measure should use the same validated order population.

A controlled pattern can use a validated list of Order IDs:

```DAX
VAR ValidOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )

RETURN
CALCULATE(
    SUM(ERPOrderLines[NetRevenue]),
    TREATAS(
        ValidOrders,
        ERPOrders[ExternalOrderID]
    )
)
```

This prevents the revenue KPI from silently including a different set of orders than the order-count KPI.

---

### 8. Validate Revenue at Order Level

Aggregate totals alone are not sufficient for diagnosing differences.

Two totals can appear close even when individual records are wrong because positive and negative differences may offset each other.

A stronger reconciliation compares individual orders.

A conceptual reconciliation table could contain:

```text
Order ID | ERP Gross | ERP Net | Marketplace Gross | Marketplace Net | Difference | Status
```

Order-level analysis helps distinguish between:

- exact matches
- tax differences
- shipping differences
- discounts
- refunds
- cancellations
- missing transactions
- timing differences
- unexplained residuals

---

### 9. Do Not Force Exact Equality

A reconciliation project should not introduce an arbitrary correction factor simply because management expects two systems to show the same number.

A difference may result from legitimate differences in:

- business definitions
- transaction timing
- source-system processing
- tax treatment
- marketplace logic
- cancellation handling
- refunds
- available data

Applying a correction factor without identifying the cause can hide a real data-quality problem.

A better reporting approach is to make the difference transparent.

For example:

```DAX
Revenue Difference =
[ERP Net Revenue]
    - [Marketplace Net Revenue]
```

The result should be described as a **source difference** until its cause has been demonstrated.

---

### 10. Make Reconciliation Transparent

A professional dashboard does not need every source to produce identical numbers.

Instead, it should make clear:

- which source provides each KPI
- which business definition is used
- which date controls the calculation
- which records are excluded
- how tax is treated
- how large the remaining difference is
- whether that difference has been explained

This leads to a more defensible analytical result than silently modifying one source until the totals match.

> **The goal of revenue reconciliation is not to manufacture identical numbers. It is to understand and communicate why the numbers differ.**

---

## Power BI Date & Filter Context

One of the most important technical lessons in this project was that a DAX measure can be mathematically correct and still return the wrong business result.

The reason is often not the calculation itself.

The reason is **filter context**.

### 1. The Same Measure Can Behave Differently

Different report pages may use different date fields or slicers.

For example:

```text
Management Page
      │
      ▼
Calendar[Date]
      │
      ▼
ERP Measures


Amazon Analysis Page
      │
      ▼
MarketplaceReport[Period]
      │
      ▼
Amazon Measures
```

A measure designed for the first page may not automatically receive the intended date filter when reused on the second page.

This can produce a result that looks like a source-data problem even though the real problem is the Power BI filter context.

> **A correct DAX formula in the wrong filter context can produce the wrong business result.**

---

### 2. Identify the Business Date First

Before writing a time-based measure, determine which date represents the business event being analyzed.

Possible dates include:

- purchase date
- ERP creation date
- payment date
- shipping date
- invoice date
- refund date
- settlement transaction date

These dates should not be used interchangeably.

For example:

```text
Order Analysis      → Purchase Date
ERP Operations      → Creation Date
Shipping Analysis   → Shipping Date
Settlement Analysis → Transaction Date
```

The correct choice depends on the analytical question.

---

### 3. A Calendar Table Does Not Solve Everything Automatically

A dedicated date dimension is an important part of a robust Power BI model.

However, simply creating a calendar table does not guarantee that every source receives the intended date filter.

Problems can still occur when:

- a fact table has multiple date columns
- a relationship is inactive
- another page uses a source-specific date field
- a measure removes filters with `ALL`
- a disconnected table controls the report
- filter propagation does not reach the required table

Therefore, the filter path must still be understood.

---

### 4. Inspect the Active Filter Context

When a KPI produces an unexpected result, the first debugging question should be:

> **Which table and column are actually filtering this visual?**

This should be checked before rewriting the measure.

Useful questions include:

- Which field is used in the date slicer?
- Which table contains that field?
- Is there an active relationship?
- Which direction does the relationship filter?
- Is the target fact table connected?
- Does the measure remove existing filters?
- Is another visual or page filter active?

This simple check can prevent unnecessary changes to otherwise correct DAX logic.

---

### 5. Explicitly Transfer the Selected Period When Necessary

In some reconciliation scenarios, the selected reporting period needs to be applied explicitly to another source.

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

This pattern makes the intended period explicit.

The `< EndDate + 1` pattern is particularly useful when the source column contains a date-time value rather than a pure date.

It includes records throughout the final selected day without requiring the time component to be removed first.

---

### 6. Different Pages May Require Different Date Sources

During reconciliation, one report page may be controlled by a central calendar while another analytical page may use the reporting period from a source-specific dataset.

In that situation, blindly reusing the same measure can produce incorrect results.

Conceptually:

```DAX
-- Management page
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])
```

while another page may require:

```DAX
-- Source comparison page
VAR StartDate =
    MIN(MonthlyReport[DateFrom])

VAR EndDate =
    MAX(MonthlyReport[DateTo])
```

The calculation being performed may be identical.

The difference is **where the selected period comes from**.

This was an important debugging lesson:

> **Before changing a calculation, verify whether the measure is reading the correct filter context.**

---

### 7. Use TREATAS for Controlled Cross-Table Filtering

Date context was not the only filter-context challenge.

After determining the valid Amazon order population, those Order IDs needed to filter the corresponding ERP records.

A controlled pattern was:

```DAX
VAR ValidOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )

RETURN
CALCULATE(
    [ERP Net Revenue],
    TREATAS(
        ValidOrders,
        ERPOrders[ExternalOrderID]
    )
)
```

`TREATAS` allows values from one table to be applied as a filter to another column without creating an additional physical relationship.

This can be useful for reconciliation models, but only after validating:

- identifier compatibility
- data grain
- duplicate behavior
- intended filter direction
- business definition

---

### 8. Debug Context Before Debugging Arithmetic

When a KPI is unexpectedly high, low, blank, or inconsistent across pages, a useful troubleshooting sequence is:

```text
Unexpected KPI
      │
      ▼
Check Slicer / Page Filters
      │
      ▼
Identify Date Source
      │
      ▼
Check Relationships
      │
      ▼
Check Filter Propagation
      │
      ▼
Check Measure Context
      │
      ▼
Only Then Check Arithmetic
```

This order matters.

Changing arithmetic before understanding context can create additional errors while hiding the original problem.

---

### 9. Build Small Control Measures

Complex reconciliation measures should be validated using smaller control measures.

Examples include:

```DAX
Control Order Count =
DISTINCTCOUNT(AmazonOrders[OrderID])
```

or:

```DAX
Control Revenue =
SUM(ERPOrderLines[NetRevenue])
```

These simple measures help answer one question at a time.

For example:

- Is the period correct?
- Is the order population correct?
- Are cancellations excluded?
- Is the marketplace filter working?
- Does the Order ID transfer work?

Once each component is validated, the final KPI becomes easier to trust and maintain.

---

### 10. General Debugging Principle

A useful rule from this project is:

> **Do not immediately rewrite a measure because the result looks wrong. First determine which data and filters the measure is actually seeing.**

For reconciliation work, DAX debugging is not only about formulas.

It is also about understanding:

- evaluation context
- relationships
- filter propagation
- date semantics
- data grain
- business definitions

This distinction can turn what appears to be a complicated calculation problem into a much simpler model or filter-context problem.
