# Troubleshooting Guide

Reconciliation problems are often caused by a combination of business definitions, source-system behavior, data modeling, and filter context.

The most effective troubleshooting approach is therefore not to change several calculations at once.

Instead:

> **Define one question, test one hypothesis, evaluate the evidence, and only then continue.**

This guide provides a reusable troubleshooting process for Power BI reconciliation projects.

---

## 1. Start With the Business Question

Before debugging Power BI, define what the KPI is supposed to represent.

For example:

```text
"Orders"
```

is not a complete definition.

A better definition is:

```text
Distinct non-cancelled Amazon orders
based on purchase date
for the selected marketplace
and reporting period.
```

Likewise:

```text
"Revenue"
```

should specify:

- gross or net
- VAT treatment
- order population
- marketplace
- reporting period
- currency
- refund treatment

Without a clear definition, it is impossible to determine whether the result is actually wrong.

---

## 2. Difference Does Not Automatically Mean Missing Data

Suppose:

```text
System A Orders = 1,000
System B Orders =   970
Difference       =    30
```

This proves only that the systems differ by 30 under the current definitions.

It does not prove that 30 orders are missing.

Possible explanations include:

- cancelled orders
- different reporting periods
- different marketplaces
- settlement timing
- duplicate records
- split transactions
- identifier variations
- different source refresh times
- different business definitions

During investigation, use a neutral description such as:

> **Source difference**

Only assign a cause after the evidence supports it.

---

## 3. Troubleshooting Order Differences

When order counts do not match, check the following sequence:

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

Do not begin with individual orders if the high-level populations have not yet been aligned.

---

## 4. Check the Reporting Period

Ask:

- Are both sources using the same period?
- Which date controls each source?
- Is one source using purchase date?
- Is another using creation date?
- Is settlement based on transaction date?
- Does a date-time field include times?

A common mistake is:

```text
Purchase Month = Settlement Month
```

This is not always valid.

Marketplace financial transactions can occur later than the original purchase.

Therefore:

> **Purchase month ≠ Settlement month**

---

## 5. Check the Order Definition

Two systems can both display **Orders** while using different rules.

Check whether each source includes:

- cancelled orders
- pending orders
- shipped orders
- refunded orders
- test orders
- internal transactions

Before comparing totals, document what counts as an order in each system.

---

## 6. Check Row Count vs. Distinct Order Count

A transaction table can contain multiple rows for one order.

Therefore:

```DAX
COUNTROWS(Transactions)
```

and:

```DAX
DISTINCTCOUNT(Transactions[OrderID])
```

answer different questions.

Always understand the table grain first.

> **One row is not automatically one order.**

---

## 7. Check Duplicate Identifiers

A repeated Order ID may be:

- a legitimate order-line repetition
- a split transaction
- a partial shipment
- a refund
- an adjustment
- an actual duplicate

Do not immediately remove duplicates.

First determine why they exist.

---

## 8. Check Blank or Invalid Identifiers

Before matching, quantify:

- blank Order IDs
- null identifiers
- malformed identifiers
- placeholder values
- unexpected formats

Records without a usable identifier cannot participate reliably in exact matching.

They should remain visible as a data-quality issue.

---

## 9. Check Marketplace Scope

A major reconciliation mistake is comparing different marketplace populations.

For example:

```text
Source A → All Amazon Marketplaces
Source B → Amazon.de Only
```

The resulting difference is not evidence of missing orders.

Check the dedicated:

- marketplace field
- sales-channel field

Do not automatically use shipping country as marketplace.

> **Shipping destination does not prove marketplace origin.**

---

## 10. Check Settlement Timing

If an ERP order cannot be found in the settlement file for the purchase month, do not immediately classify it as missing.

Use:

```text
Purchase Period
      ↓
Search Same Settlement Period
      ↓
Search Later Settlement Periods
      ↓
Still Not Found?
      ↓
Residual Investigation
```

The appropriate search horizon depends on the available data and business process.

---

## 11. Check Identifier Variations

Structured identifiers may appear with variations such as:

```text
ORDER-123
ORDER-123_1
ORDER-123_2
```

These may represent:

- split transactions
- partial processing
- adjustments
- partial shipments
- source-specific logic

Normalization can be useful during investigation.

However:

> **Do not permanently remove identifier suffixes until their business meaning has been validated.**

---

## 12. Troubleshooting Revenue Differences

When revenue does not match, use this sequence:

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

Revenue should not be debugged independently from the underlying order population.

---

## 13. Check Gross vs. Net

One source may report:

```text
Gross Revenue
```

while another reports:

```text
Net Revenue
```

A difference is expected if VAT has not been aligned.

Before comparing financial values, identify whether each source is:

- tax-inclusive
- tax-exclusive
- storing VAT separately
- storing VAT as positive or negative

---

## 14. Check VAT Treatment

VAT can be represented differently across systems.

Possible patterns include:

```text
Gross amount includes VAT
Net amount excludes VAT
VAT stored separately
VAT stored as negative amount
Marketplace-withheld tax stored separately
```

Do not assume the sign or treatment from the column name alone.

Validate it against the actual source behavior.

---

## 15. Check Sales Components for Double Counting

If a source contains several sales-related metrics, determine whether they overlap.

For example:

```text
Organic Sales
PPC Sales
Sponsored Sales
```

Do not automatically add every field containing the word **Sales**.

One metric may already be included in another.

---

## 16. Check Refund Timing

An order can be purchased in one month and refunded later.

Therefore:

```text
Purchase Period ≠ Refund Period
```

Check:

- original Order ID
- purchase date
- refund date
- refund amount
- VAT adjustment
- reporting-period definition

---

## 17. Check Fees and Payout

Marketplace fees should not automatically be treated as reductions of sales revenue when comparing revenue KPIs.

Settlement data may contain:

- selling fees
- fulfilment fees
- transaction fees
- adjustments
- shipping charges

Likewise:

> **Payout is not revenue.**

A payout represents a later financial settlement process.

---

## 18. Check Currency

For international marketplace data, verify:

- source currency
- ERP currency
- reporting currency
- exchange rate
- exchange-rate date
- rounding

Never aggregate multiple currencies as though they were the same unit.

---

## 19. Check Rounding

Small amount differences may result from different rounding methods.

For example:

```text
Round Each Line
      ↓
Sum
```

versus:

```text
Sum Exact Values
      ↓
Round Total
```

Before classifying a small discrepancy as an error, understand the calculation grain.

---

## 20. Troubleshooting Power BI Measures

When a Power BI measure looks wrong, check:

```text
Slicer / Page Filters
      ↓
Visual Filters
      ↓
Date Source
      ↓
Relationships
      ↓
Filter Direction
      ↓
Data Grain
      ↓
Business Population
      ↓
DAX Context
      ↓
Source Data
```

Do not start by rewriting the entire measure.

---

## 21. Check the Slicer

Ask:

- Which field is used?
- Which table contains the field?
- Is the slicer actually filtering the target table?
- Is the selected period what you expect?
- Are multiple values selected?

A slicer that looks correct visually can still control the wrong source for a particular measure.

---

## 22. Check Page and Visual Filters

Power BI can apply filters at several levels:

```text
Report
Page
Visual
Slicer
Relationship
Measure
```

An unexpected KPI may be caused by a filter that is not immediately visible in the measure.

Check the Filters pane before changing DAX.

---

## 23. Check the Date Source

This was an especially important issue in this project.

One page may use:

```text
Calendar[Date]
```

while another page may use:

```text
MonthlyReport[DateFrom]
```

A measure reading the wrong period source can produce an incorrect business result even if the formula itself is valid.

---

## 24. Check Relationships

Verify:

- relationship exists
- relationship is active
- cardinality is correct
- filter direction is intentional
- no ambiguous path exists
- no accidental many-to-many relationship exists

Do not change a relationship only to make one visual produce the expected number.

Understand the model consequence first.

---

## 25. Check ALL

If a measure uses:

```DAX
ALL(...)
```

determine exactly which filter is being removed.

Compare:

```DAX
ALL(Table[Date])
```

with:

```DAX
ALL(Table)
```

Removing the entire table context can have much wider effects than removing one column filter.

---

## 26. Check TREATAS

If `TREATAS` is used, validate:

- source values
- target values
- identifier compatibility
- blanks
- duplicates
- marketplace
- status population

`TREATAS` transfers filters.

It does not validate the business meaning of the identifiers.

---

## 27. Check SELECTEDVALUE

A measure using:

```DAX
SELECTEDVALUE(...)
```

can return blank if:

- nothing is selected
- multiple values are selected
- the visual contains several values

If channel-switch logic behaves unexpectedly, verify the selection context first.

---

## 28. Check Blank vs. Zero

Do not immediately convert every blank result into zero.

A blank may reveal:

- no matching records
- missing relationship
- incorrect context
- missing selection
- no source data

During debugging, blanks contain useful information.

---

## 29. Check the Visual Total

A total row is often a new evaluation of the measure in total context.

It is not necessarily:

```text
Visible Row 1
+ Visible Row 2
+ Visible Row 3
```

This matters especially for:

- distinct counts
- ratios
- conditional logic
- iterators

If row values look correct but the total does not, investigate the total context.

---

## 30. Build Control Measures

When a complex measure fails, create simple controls.

For example:

```DAX
Control Orders =
DISTINCTCOUNT(AmazonOrders[OrderID])
```

```DAX
Control Revenue =
SUM(ERPOrderLines[NetRevenue])
```

Then add business rules one at a time.

Conceptually:

```text
Raw Count
   ↓
Date Filter
   ↓
Status Filter
   ↓
Marketplace Filter
   ↓
Cross-Table Filter
   ↓
Final KPI
```

This makes the point of failure easier to identify.

---

## 31. Do Not Change Multiple Variables Simultaneously

Avoid this troubleshooting pattern:

```text
Change DAX
Change Relationship
Change Date
Change Status
Refresh
Result Looks Better
```

You cannot reliably determine which change solved the problem.

Use:

```text
One Question
     ↓
One Hypothesis
     ↓
One Test
     ↓
One Result
     ↓
Next Step
```

---

## 32. Validate Individual Records

After aggregate controls have reduced the problem, inspect individual cases.

Useful fields include:

- Order ID
- purchase date
- creation date
- payment date
- shipping date
- status
- cancellation timestamp
- marketplace
- shipping country
- gross amount
- net amount

This can identify the actual business reason behind a discrepancy.

---

## 33. Do Not Generalize From One Record

One matching order confirms that specific test case.

It does not prove that every order behaves the same way.

Likewise, one problematic order does not prove that the entire source is wrong.

State conclusions according to the available evidence.

---

## 34. Use Evidence Levels

### Confirmed

Directly demonstrated by the relevant data or source-system record.

### Empirically Supported

Supported by repeated observations but not formally documented by the source system.

### Hypothesis

A plausible explanation requiring further testing.

### Unresolved

The available evidence does not support a reliable explanation.

This prevents hypotheses from becoming undocumented facts.

---

## 35. Do Not Hide Differences With a Correction Factor

A correction factor can make totals look identical without explaining why they were different.

It may hide:

- incorrect filters
- missing transactions
- status differences
- VAT differences
- marketplace mismatches
- source-system problems

Use correction logic only when there is a documented business rule behind it.

Never use it merely to manufacture equality.

---

## 36. Common Mistakes

### Mistake: Difference = Missing Data

**Better:** call it a source difference until the cause is demonstrated.

### Mistake: Row Count = Order Count

**Better:** understand the grain and business key.

### Mistake: Same KPI Name = Same Definition

**Better:** document the underlying business logic.

### Mistake: Shipping Country = Marketplace

**Better:** use the dedicated marketplace or sales-channel field.

### Mistake: Purchase Month = Settlement Month

**Better:** search the appropriate later financial periods.

### Mistake: Payout = Revenue

**Better:** compare equivalent financial concepts.

### Mistake: Identifier Suffix = Safe to Delete

**Better:** validate the business meaning first.

### Mistake: One Matching Record = Dataset Validated

**Better:** limit the conclusion to the evidence.

### Mistake: Correct Total = Correct Data

**Better:** also validate record-level differences.

### Mistake: Wrong KPI = Wrong Formula

**Better:** inspect filter context and the model first.

---

## 37. Order Troubleshooting Decision Tree

```text
Orders Differ
     │
     ▼
Same Reporting Period?
     │
     ├── No → Align Period
     │
     ▼
Same Business Definition?
     │
     ├── No → Align Definition
     │
     ▼
Same Status Rules?
     │
     ├── No → Align Status
     │
     ▼
Distinct IDs Compared?
     │
     ├── No → Validate Grain
     │
     ▼
Same Marketplace?
     │
     ├── No → Align Marketplace
     │
     ▼
Exact IDs Match?
     │
     ├── No → Check Timing / Identifier Variations
     │
     ▼
Residual Cases
     │
     ▼
Manual Validation
```

---

## 38. Revenue Troubleshooting Decision Tree

```text
Revenue Differs
     │
     ▼
Same Order Population?
     │
     ├── No → Align Population
     │
     ▼
Same Period?
     │
     ├── No → Align Period
     │
     ▼
Same Marketplace / Currency?
     │
     ├── No → Align Scope
     │
     ▼
Gross vs. Net Aligned?
     │
     ├── No → Align Definition
     │
     ▼
VAT Aligned?
     │
     ├── No → Validate Tax Logic
     │
     ▼
Refunds / Discounts / Shipping?
     │
     ▼
Fees / Settlement Effects?
     │
     ▼
Order-Level Comparison
     │
     ▼
Residual Difference
```

---

## 39. Power BI Troubleshooting Decision Tree

```text
KPI Looks Wrong
     │
     ▼
Check Slicer
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
Build Control Measure
     │
     ▼
Check DAX Context
     │
     ▼
Check Source Data
```

---

## 40. Recommended Troubleshooting Log

For larger projects, keep a simple investigation log.

For example:

```text
Date
Issue
Expected Result
Actual Result
Hypothesis
Test Performed
Evidence
Conclusion
Status
```

Possible statuses:

```text
OPEN
CONFIRMED
RESOLVED
UNRESOLVED
```

This prevents the same issue from being investigated repeatedly and creates an audit trail.

---

## 41. Stop Conditions

Not every discrepancy should be investigated indefinitely.

A useful stopping point may be reached when:

- the business definitions are aligned
- the major discrepancy drivers are explained
- remaining cases are quantified
- remaining cases are documented
- available source data has been exhausted
- further investigation requires another system or business owner

At that point:

```text
Unresolved
```

is a legitimate result.

Do not invent certainty simply to close the analysis.

---

## 42. Reusable Troubleshooting Method

A professional troubleshooting process can be summarized as:

```text
1. Define the expected business result
              ↓
2. Identify the source population
              ↓
3. Check reporting period
              ↓
4. Check data grain
              ↓
5. Check status logic
              ↓
6. Check marketplace / currency
              ↓
7. Check identifiers
              ↓
8. Check model relationships
              ↓
9. Check filter context
              ↓
10. Build small control measures
              ↓
11. Compare individual records
              ↓
12. Evaluate evidence
              ↓
13. Document conclusion
              ↓
14. Keep unresolved cases visible
```

---

## Final Troubleshooting Principle

The most useful question during reconciliation is often not:

> **How can I make these numbers match?**

It is:

> **What exactly does each number represent, and what evidence explains the difference?**

Once the business meaning, population, period, grain, marketplace, financial definition, and Power BI context are understood, many apparent data problems become much easier to diagnose.

> **Troubleshooting should reduce uncertainty through evidence, not hide uncertainty through adjustments.**
