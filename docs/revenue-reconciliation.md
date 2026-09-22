# Revenue Reconciliation Methodology

Reconciling revenue across e-commerce, ERP, reporting, and settlement systems is more complex than comparing two monetary totals.

Different systems may report different financial values for the same business activity while all calculations are technically correct.

The first question should therefore not be:

> **Why are the numbers different?**

The first question should be:

> **Are we actually comparing the same financial concept, population, period, marketplace, and currency?**

---

## 1. Revenue Reconciliation Workflow

A structured revenue reconciliation can follow this sequence:

```text
Define Order Population
        │
        ▼
Define Reporting Period
        │
        ▼
Align Marketplace
        │
        ▼
Align Currency
        │
        ▼
Define Gross / Net
        │
        ▼
Validate VAT
        │
        ▼
Check Shipping / Discounts
        │
        ▼
Check Refunds / Adjustments
        │
        ▼
Separate Fees / Payout
        │
        ▼
Compare Order-Level Amounts
        │
        ▼
Analyze Residual Difference
```

The sequence matters.

A revenue comparison is difficult to interpret if the underlying order populations are already different.

---

## 2. Define the Financial Metric First

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
- cost of goods
- profit

These values should not be treated as interchangeable.

Conceptually:

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

The exact calculation depends on the source and accounting definition.

The important distinction is:

> **Revenue ≠ Settlement ≠ Payout ≠ Profit**

---

## 3. Understand Every Source Independently

Before comparing financial values, determine what each source actually reports.

A monthly marketplace reporting source may contain fields such as:

- Organic Sales
- PPC Sales
- VAT
- Refunds
- Fees
- Shipping Costs
- Estimated Payout
- Net Profit

An ERP/WaWi source may instead contain:

- gross order value
- net order value
- invoice value
- open amount
- payment information
- order-line revenue

A settlement source may contain:

- product sales
- product VAT
- shipping credits
- discounts
- refunds
- selling fees
- fulfilment fees
- transaction totals

The field names alone are not sufficient evidence that two values are comparable.

---

## 4. Gross Revenue vs. Net Revenue

A fundamental distinction is gross versus net revenue.

Conceptually:

```text
Gross Revenue
      │
      ├── VAT
      ▼
Net Revenue
```

A simplified relationship may be:

```text
Gross Revenue = Net Revenue + VAT
```

However, real marketplace data can contain additional effects such as:

- different VAT rates
- tax adjustments
- shipping VAT
- promotional discounts
- refunds
- marketplace tax handling
- rounding

Therefore, this relationship should be used as a plausibility check rather than blindly assumed for every transaction.

---

## 5. Validate VAT Treatment

Before comparing net revenue, determine how VAT is represented in each source.

Possible representations include:

```text
Gross value with VAT included
Net value without VAT
VAT stored separately as positive value
VAT stored separately as negative value
Marketplace-withheld tax
Tax adjustments
```

A source-specific measure may therefore require different arithmetic.

For example, if a monthly reporting export stores VAT as a negative value:

```DAX
Net Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
    + SUM(MonthlyReport[VAT])
```

This works only if the source semantics have first been confirmed.

> **Never copy a revenue formula without validating how the source represents tax.**

---

## 6. Avoid Double Counting Sales Components

Reporting tools may contain several sales-related fields.

For example:

```text
Organic Sales
PPC Sales
Sponsored Product Sales
Sponsored Display Sales
```

Before adding them together, determine whether one metric already contains another component.

Otherwise, revenue can be double counted.

The correct process is:

```text
Inspect Source Definition
        ↓
Determine Component Relationship
        ↓
Only Then Build Measure
```

A plausible-looking total does not prove that the components were combined correctly.

---

## 7. Compare Equivalent Financial Concepts

Examples of unsafe comparisons include:

```text
ERP Net Revenue
        vs.
Marketplace Payout
```

```text
Gross Order Value
        vs.
Net Revenue
```

```text
Sales Revenue
        vs.
Settlement Total
```

```text
Purchase-Month Revenue
        vs.
Settlement-Month Transactions
```

A stronger comparison is:

```text
Source A Net Revenue
        vs.
Source B Net Revenue
```

with aligned:

- order population
- reporting period
- marketplace
- status logic
- currency
- VAT treatment
- revenue components

Only then does the remaining difference become analytically meaningful.

---

## 8. Use the Same Order Population

Revenue should be calculated for the same intended order population as the related order KPI.

For example:

```text
Valid Amazon Orders
        │
        ├──────────────► Order Count
        │
        └──────────────► ERP Net Revenue
```

If cancelled orders are excluded from the order KPI but remain included in the revenue KPI, the two KPIs describe different populations.

A controlled DAX pattern can be:

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

The exact tables and fields will vary by model.

The principle remains the same:

> **Order count and revenue should use compatible business populations.**

---

## 9. Validate Marketplace Scope

Revenue reconciliation should compare the same marketplace scope.

For example:

```text
All Amazon Marketplaces
```

should not be compared directly with:

```text
Amazon.de Only
```

without making the scope difference explicit.

Marketplace should be identified using the appropriate sales-channel or marketplace field.

Shipping country should not be used as a substitute.

---

## 10. Validate Currency

International marketplace activity may involve multiple currencies.

Before aggregating monetary values, verify:

- transaction currency
- reporting currency
- ERP currency
- exchange-rate source
- exchange-rate date
- conversion logic
- rounding rules

Conceptually:

```text
100 EUR + 100 SEK
```

must not simply become:

```text
200 Revenue
```

unless a valid currency-conversion process has been applied.

---

## 11. Separate Sales Revenue from Marketplace Fees

Settlement exports can contain:

- selling fees
- fulfilment fees
- transaction fees
- shipping-related charges
- storage fees
- other adjustments

These values explain how money moves between the marketplace and the seller.

They do not automatically redefine the original sales revenue.

For example:

```text
Customer Purchase
      │
      ▼
Sales Revenue
      │
      ├── Marketplace Fees
      ├── Fulfilment Fees
      ├── Adjustments
      ▼
Settlement / Payout
```

Therefore, marketplace fees should not automatically be deducted when the objective is to compare sales revenue.

---

## 12. Payout Is Not Revenue

A payout is the result of a financial settlement process.

It may include:

- sales
- refunds
- fees
- tax effects
- fulfilment charges
- reimbursements
- adjustments
- previous-period activity

Therefore:

```text
Marketplace Payout
```

is generally not equivalent to:

```text
ERP Sales Revenue
```

Comparing them directly can create a large apparent discrepancy even when both values are correct.

---

## 13. Treat Refunds Separately

Refunds introduce another timing problem.

An order may be purchased in one month and refunded later.

Therefore:

```text
Purchase Period ≠ Refund Period
```

A refund analysis should consider:

- original Order ID
- original purchase date
- refund transaction date
- refunded amount
- VAT adjustment
- marketplace
- reporting-period definition

A purchase-month revenue KPI and a settlement-period financial KPI may legitimately differ because they answer different questions.

---

## 14. Consider Promotional Discounts

Promotional discounts can affect the amount visible in different systems.

Depending on the source, discounts may be:

- already included in sales
- stored separately
- represented as negative transactions
- split between product and tax components

Before adjusting revenue, determine how the source represents the discount.

Otherwise, the discount may be deducted twice.

---

## 15. Consider Shipping Components

Shipping can also be represented differently.

Possible fields include:

- shipping revenue
- shipping credit
- shipping VAT
- shipping cost
- fulfilment fee

These are not the same financial concept.

For example:

```text
Customer Shipping Charge
```

and:

```text
Seller Shipping Cost
```

should not be treated as equivalent.

Shipping components should therefore be classified before they are included in a reconciliation formula.

---

## 16. Settlement Transaction Total Requires Care

A settlement field called:

```text
Total
```

may look like an ideal comparison value.

However, settlement totals can include multiple financial components beyond sales.

Before using such a field, determine whether it contains:

- sales
- VAT
- fees
- refunds
- adjustments
- shipping
- fulfilment charges

A field name such as **Total** does not define its business meaning.

---

## 17. Validate Revenue at Order Level

Aggregate totals alone are not sufficient for diagnosing financial differences.

Two totals can appear close even when individual orders differ.

For example:

```text
Order A Difference   +50
Order B Difference   -50
-------------------------
Total Difference       0
```

The aggregate appears perfect, but two discrepancies still exist.

A stronger reconciliation compares individual orders.

Conceptually:

```text
Order ID
ERP Gross
ERP Net
Marketplace Gross
Marketplace Net
VAT
Difference
Match Status
```

---

## 18. Use Exact Matches as Evidence

When comparing order-level amounts, exact or near-exact matches can provide strong evidence about field semantics.

For example, if many individual orders show:

```text
ERP Gross
≈
Marketplace Product Sales + Product VAT
```

that may provide empirical evidence that the compared fields represent similar customer-facing amounts.

However:

> **Empirical matching is not the same as formal source-system documentation.**

The conclusion should reflect the strength of the evidence.

---

## 19. Use Tolerance Carefully

Small differences can result from:

- rounding
- tax rounding
- line-level calculation
- currency conversion
- decimal precision

A reconciliation may therefore use a documented tolerance.

Conceptually:

```text
ABS(SourceA - SourceB) <= Tolerance
```

For example:

```DAX
Amount Match =
IF(
    ABS([Source A Amount] - [Source B Amount]) <= 0.01,
    "Matched",
    "Investigate"
)
```

The tolerance should have a defensible reason.

It should not be increased simply to make more records match.

---

## 20. Understand Rounding Grain

Rounding can occur at different levels.

For example:

```text
Method A
Round each order line
        ↓
Sum rounded lines
```

versus:

```text
Method B
Sum precise lines
        ↓
Round final total
```

These methods can produce slightly different results.

Therefore, small discrepancies should be investigated before being classified as data errors.

---

## 21. Distinguish Amount Difference from Missing Transaction

An order may exist in both systems while the amounts differ.

This is different from an Order ID being absent.

A reconciliation should therefore distinguish:

```text
Order Match Status
```

from:

```text
Amount Match Status
```

For example:

```text
Order ID Found = Yes
Amount Match   = No
```

requires financial investigation, not order-existence investigation.

---

## 22. Revenue Difference KPI

Once comparable revenue measures have been validated, the remaining source difference can be shown transparently.

For example:

```DAX
Revenue Difference =
[ERP Net Revenue]
    - [Marketplace Net Revenue]
```

The result should initially be described as:

> **Revenue source difference**

until its cause has been demonstrated.

It should not automatically be described as missing or incorrect revenue.

---

## 23. Percentage Difference

A percentage can provide useful context for the size of a discrepancy.

Conceptually:

```DAX
Revenue Difference % =
DIVIDE(
    [Revenue Difference],
    [Marketplace Net Revenue]
)
```

However, the percentage does not explain the cause.

It only describes the relative size of the difference.

---

## 24. Do Not Use a Correction Factor Without Evidence

A correction factor can make two systems display identical totals.

For example:

```text
Source A Revenue × Adjustment Factor
        =
Source B Revenue
```

This may improve visual agreement while hiding the underlying cause.

Possible hidden problems include:

- wrong order population
- wrong marketplace
- incorrect VAT treatment
- refunds
- missing transactions
- date-context problems
- incorrect source definitions

A correction factor should only be used when there is a documented and defensible business rule.

It should not be used simply to force equality.

---

## 25. Validate Aggregate and Record-Level Results

A strong revenue validation should operate at multiple levels.

### Level 1 — Source Definition

What does each field mean?

### Level 2 — Population

Are the same orders included?

### Level 3 — Financial Definition

Are both values gross, net, or another comparable concept?

### Level 4 — Record Level

Do individual orders match?

### Level 5 — Aggregate Level

Do the totals reconcile after the earlier levels are validated?

The order matters.

A plausible aggregate total should not replace the earlier validation steps.

---

## 26. Revenue Troubleshooting Decision Tree

When revenue does not match, check:

```text
Order Population
      ↓
Reporting Period
      ↓
Marketplace
      ↓
Currency
      ↓
Gross vs. Net
      ↓
VAT Treatment
      ↓
Cancellations
      ↓
Refunds
      ↓
Discounts
      ↓
Shipping Components
      ↓
Fees / Adjustments
      ↓
Order-Level Amounts
      ↓
Rounding
      ↓
Residual Difference
```

This sequence prevents unrelated financial concepts from being mixed together.

---

## 27. Useful Revenue Controls

Simple control measures can help isolate problems.

For example:

```DAX
ERP Net Revenue Control =
SUM(ERPOrderLines[NetRevenue])
```

```DAX
Marketplace Gross Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
```

A source-specific net-like measure might be:

```DAX
Marketplace Net Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
    + SUM(MonthlyReport[VAT])
```

Again, this final pattern is valid only when the source stores VAT as the expected signed value and the sales components have been validated.

---

## 28. Keep Financial Definitions Documented

Every important revenue measure should have a documented definition.

A useful KPI definition can include:

```text
KPI Name
Business Purpose
Source
Order Population
Date Definition
Marketplace Scope
Gross / Net Definition
VAT Treatment
Refund Treatment
Currency
Exclusions
Known Limitations
```

This makes the calculation understandable to analysts who did not build the original report.

---

## 29. Make Source Differences Visible

A professional dashboard does not require every system to show identical revenue.

Instead, it should make clear:

- which source provides each KPI
- which financial definition is used
- which date controls the calculation
- which marketplace is included
- which orders are excluded
- how VAT is treated
- how refunds are treated
- how large the remaining difference is
- whether the difference has been explained

This is more defensible than silently modifying one source until the totals match.

---

## 30. Example Revenue Reconciliation Output

A dedicated reconciliation page could contain:

```text
Marketplace Net Revenue
ERP Net Revenue
Revenue Difference
Revenue Difference %

Matched Orders
Orders with Amount Difference
Refund-Related Differences
Timing Differences
Unresolved Amount Differences
```

This allows operational reporting and data-quality reporting to remain separate.

---

## 31. Evidence Levels for Financial Findings

Financial conclusions should also reflect evidence strength.

### Confirmed

Directly supported by source documentation or validated source-system records.

### Empirically Supported

Strongly supported by repeated transaction-level matching.

### Hypothesis

A plausible explanation that still requires testing.

### Unresolved

The available evidence is insufficient to explain the difference.

This prevents an analytical assumption from being presented as an accounting fact.

---

## 32. Reusable Revenue Reconciliation Workflow

A reusable process is:

```text
1. Define the revenue question
              ↓
2. Define the order population
              ↓
3. Define the reporting period
              ↓
4. Align marketplace
              ↓
5. Align currency
              ↓
6. Identify gross / net fields
              ↓
7. Validate VAT treatment
              ↓
8. Validate sales components
              ↓
9. Separate fees and payout
              ↓
10. Check refunds and discounts
              ↓
11. Check shipping components
              ↓
12. Compare individual orders
              ↓
13. Investigate amount differences
              ↓
14. Compare aggregate totals
              ↓
15. Document residual differences
```

---

## Final Takeaway

Revenue reconciliation is not simply:

```text
Source A - Source B
```

A meaningful comparison requires alignment of:

```text
Order Population
      ↓
Reporting Period
      ↓
Marketplace
      ↓
Currency
      ↓
Financial Definition
      ↓
VAT
      ↓
Refunds / Discounts
      ↓
Transaction Timing
      ↓
Record-Level Validation
      ↓
Residual Difference
```

The goal is not to manufacture identical numbers.

The goal is to determine whether the compared values represent the same business concept and to explain the remaining difference as far as the available evidence allows.

> **A trustworthy revenue reconciliation explains the meaning of the numbers before it explains the difference between them.**
