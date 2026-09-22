# DAX & Filter Context

One of the most important technical lessons from this project was that a DAX measure can be mathematically correct and still return the wrong business result.

The problem is often not the arithmetic.

The problem is **filter context**.

A measure only calculates over the rows that are visible in its current evaluation context. That context can be influenced by:

- slicers
- page filters
- visual filters
- relationships
- filter direction
- date tables
- other dimensions
- `CALCULATE`
- `FILTER`
- `ALL`
- `TREATAS`
- the visual in which the measure is used

Therefore:

> **Before changing a DAX formula because the result looks wrong, determine which data and filters the measure is actually seeing.**

---

## 1. Why Filter Context Matters

Imagine a measure that correctly calculates Amazon orders.

On one report page, the selected period may come from:

```text
Calendar[Date]
```

On another page, the selected period may come from:

```text
MonthlyReport[DateFrom]
MonthlyReport[DateTo]
```

The same calculation can therefore behave differently depending on which table provides the active filter.

Conceptually:

```text
Management Page
      │
      ▼
Calendar[Date]
      │
      ▼
Amazon Measure
```

versus:

```text
Amazon Analysis Page
      │
      ▼
MonthlyReport[Period]
      │
      ▼
Amazon Measure
```

If the second filter does not propagate to the required fact table, the measure may evaluate a much larger or completely different population.

---

## 2. Filter Context Is Part of the Business Logic

Filter context is not only a technical Power BI concept.

It determines which business records belong to a KPI.

For example:

```text
Selected Month
      ↓
Selected Marketplace
      ↓
Valid Order Status
      ↓
Visible Order Population
      ↓
KPI Result
```

If one of these filters is missing or comes from the wrong table, the KPI can represent the wrong business population.

Therefore:

> **Filter context is part of the KPI definition.**

---

## 3. Identify the Business Date First

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

Conceptually:

```text
Order Analysis      → Purchase Date
ERP Operations      → Creation Date
Shipping Analysis   → Shipping Date
Refund Analysis     → Refund Date
Settlement Analysis → Transaction Date
```

The correct date depends on the analytical question.

---

## 4. A Calendar Table Does Not Solve Everything Automatically

A dedicated date dimension is an important part of a robust Power BI model.

However, simply creating a calendar table does not guarantee that every source receives the intended date filter.

Problems can still occur when:

- a fact table has multiple date columns
- a relationship is inactive
- another report page uses a source-specific date field
- a measure removes filters
- a disconnected table controls the report
- filter propagation does not reach the required table
- a source uses a different business date

Therefore, the actual filter path must still be understood.

---

## 5. Inspect the Active Filter Context

When a KPI produces an unexpected result, the first debugging question should be:

> **Which table and column are actually filtering this visual?**

Check:

- Which field is used in the slicer?
- Which table contains that field?
- Is there an active relationship?
- Which direction does the relationship filter?
- Does the filter reach the required fact table?
- Is another page filter active?
- Is a visual-level filter active?
- Does the measure remove filters?
- Is a disconnected table involved?

This should be checked before rewriting the calculation.

---

## 6. Row Context vs. Filter Context

DAX contains two important context concepts.

### Filter Context

Filter context determines which rows are visible to an expression.

It can come from:

- slicers
- visual axes
- filters
- relationships
- `CALCULATE`

### Row Context

Row context represents the current row during row-by-row evaluation.

It commonly appears in:

- calculated columns
- iterator functions such as `SUMX`
- `FILTER`

Understanding the distinction is important because a row being evaluated does not automatically mean that the same row is filtering every related calculation.

---

## 7. CALCULATE Changes Filter Context

`CALCULATE` is one of the most important DAX functions because it evaluates an expression in a modified filter context.

A simplified example is:

```DAX
Valid Orders =
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    AmazonOrders[OrderStatus] <> "Canceled"
)
```

Conceptually:

```text
Current Filter Context
        │
        ▼
CALCULATE
        │
        ├── Add / Modify Status Filter
        ▼
New Filter Context
        │
        ▼
DISTINCTCOUNT
```

This makes `CALCULATE` extremely powerful, but it also means that every filter argument should have a clear business purpose.

---

## 8. Build the Reporting Period Explicitly When Necessary

In some reconciliation scenarios, the selected reporting period must be transferred explicitly to another source.

A useful pattern is:

```DAX
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])

RETURN
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    FILTER(
        ALL(AmazonOrders[PurchaseDate]),
        AmazonOrders[PurchaseDate] >= StartDate
            &&
        AmazonOrders[PurchaseDate] < EndDate + 1
    )
)
```

This makes the intended reporting period explicit.

---

## 9. Why `< EndDate + 1` Can Be Useful

A source column may contain a date-time rather than a pure date.

For example:

```text
2026-02-28 00:05:00
2026-02-28 14:30:00
2026-02-28 23:59:59
```

A condition such as:

```DAX
PurchaseDate <= EndDate
```

can behave unexpectedly if `EndDate` represents midnight at the beginning of the final day.

A useful pattern is:

```DAX
PurchaseDate < EndDate + 1
```

Conceptually:

```text
StartDate <= PurchaseDate < Day After EndDate
```

This includes the complete final selected day.

---

## 10. Different Pages May Require Different Period Sources

A measure designed for a management page may use:

```DAX
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])
```

A source-comparison page may instead be controlled by a monthly reporting table:

```DAX
VAR StartDate =
    MIN(MonthlyReport[DateFrom])

VAR EndDate =
    MAX(MonthlyReport[DateTo])
```

The business calculation can otherwise remain similar.

The important difference is:

> **Where does the selected period come from?**

This can explain why a measure works correctly on one page and incorrectly on another.

---

## 11. Reusing Measures Across Pages Requires Validation

Reusing measures is generally desirable.

However, reuse is safe only when the measure receives the intended context on every page.

Before reusing a measure, check:

```text
Same Date Source?
Same Marketplace Context?
Same Status Context?
Same Business Population?
Same Relationships?
```

If the answer is no, a page-specific measure or explicit filter transfer may be required.

This is preferable to assuming that one measure automatically has the same meaning everywhere.

---

## 12. Use TREATAS for Controlled Filter Transfer

In reconciliation models, a validated set of business identifiers may need to filter another table.

`TREATAS` can apply values from one table expression as filters to another column.

A simplified pattern is:

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

Conceptually:

```text
Amazon Order IDs
       │
       ▼
    TREATAS
       │
       ▼
ERP External Order IDs
       │
       ▼
ERP Revenue
```

This can be useful when adding another physical relationship would create model complexity or unwanted many-to-many behavior.

---

## 13. Validate TREATAS Before Using It

`TREATAS` does not prove that two columns represent the same business key.

Before using it, verify:

- both fields represent compatible identifiers
- formatting is compatible
- blanks are understood
- duplicates are understood
- table grain is known
- marketplace scope is aligned
- status rules are aligned

A technically successful filter transfer can still represent the wrong business relationship.

---

## 14. ALL Can Remove More Context Than Expected

`ALL` is useful for removing filters before applying new logic.

For example:

```DAX
FILTER(
    ALL(AmazonOrders[PurchaseDate]),
    AmazonOrders[PurchaseDate] >= StartDate
        &&
    AmazonOrders[PurchaseDate] < EndDate + 1
)
```

Here the existing filter on `PurchaseDate` is removed and replaced by an explicit date range.

However, `ALL` should be used carefully.

Ask:

- Which filter is being removed?
- Is only one column being cleared?
- Is an entire table being cleared?
- Could a user selection be unintentionally ignored?

The difference between:

```DAX
ALL(Table[Column])
```

and:

```DAX
ALL(Table)
```

can be significant.

---

## 15. Relationship Direction Matters

Relationships determine how filters propagate through the model.

Conceptually:

```text
Dimension
    │
    ▼
Fact Table
```

is easier to reason about than uncontrolled bidirectional filtering.

Bidirectional relationships can sometimes be necessary, but they can also create:

- ambiguous filter paths
- unexpected interactions
- difficult debugging
- incorrect totals

Filter direction should therefore be intentional rather than used simply to make a visual start working.

---

## 16. Inactive Relationships

A fact table can contain multiple date columns, while only one relationship to the date table can normally be active between the same two tables.

Conceptually:

```text
DimDate
   │
   ├── Active   → PurchaseDate
   │
   └── Inactive → ShippingDate
```

If an analysis requires the inactive relationship, a measure may intentionally activate it.

A common pattern is:

```DAX
Shipped Revenue =
CALCULATE(
    [Net Revenue],
    USERELATIONSHIP(
        DimDate[Date],
        Orders[ShippingDate]
    )
)
```

The business meaning should be clear:

```text
Revenue by Purchase Date
```

and:

```text
Revenue by Shipping Date
```

answer different questions.

---

## 17. Disconnected Tables

Some Power BI designs intentionally use disconnected tables for:

- KPI selectors
- scenario parameters
- custom period selectors
- switch logic

A disconnected table does not automatically filter a fact table.

The measure must explicitly interpret the selected value.

Therefore, when a slicer appears to have no effect, check whether its table is actually connected to the model or intentionally disconnected.

---

## 18. SELECTEDVALUE Requires Context

`SELECTEDVALUE` is useful when one value is expected in the current context.

For example:

```DAX
Selected Channel =
SELECTEDVALUE(Channel[ChannelName])
```

However, it can return blank when:

- no value is selected
- multiple values are selected
- the current visual context contains multiple values

Therefore, measures using `SELECTEDVALUE` should account for the expected selection behavior.

---

## 19. SWITCH for Business Logic

`SWITCH` can be useful when different channels require different calculation logic.

Conceptually:

```DAX
Revenue Management =
VAR SelectedChannel =
    SELECTEDVALUE(Channel[Channel])

RETURN
SWITCH(
    TRUE(),

    SelectedChannel = "Amazon",
        [Amazon Revenue],

    SelectedChannel = "eBay",
        [eBay Revenue],

    [Default Revenue]
)
```

This can keep channel-specific business rules inside one management KPI.

However, each branch still needs to use the correct:

- date context
- order population
- financial definition
- marketplace logic

---

## 20. Measure Branching

Complex calculations are easier to maintain when they are built from validated base measures.

For example:

```text
Base Net Revenue
       │
       ▼
Amazon Net Revenue
       │
       ▼
Management Revenue
       │
       ▼
Revenue Difference
```

This is often preferable to repeating the entire calculation inside every final KPI.

Benefits include:

- easier debugging
- less duplicated logic
- clearer definitions
- easier maintenance

---

## 21. Build Small Control Measures

When a complex KPI is wrong, create small controls that answer one question at a time.

Examples:

```DAX
Control Orders =
DISTINCTCOUNT(AmazonOrders[OrderID])
```

```DAX
Control Canceled Orders =
CALCULATE(
    DISTINCTCOUNT(AmazonOrders[OrderID]),
    AmazonOrders[OrderStatus] = "Canceled"
)
```

```DAX
Control Revenue =
SUM(ERPOrderLines[NetRevenue])
```

These can help determine whether the problem lies in:

- the reporting period
- the status filter
- the relationship
- the identifier transfer
- the financial calculation

---

## 22. Validate Variables Individually

Complex measures often use several variables.

For example:

```DAX
VAR StartDate = ...
VAR EndDate = ...
VAR ValidOrders = ...
VAR Revenue = ...
RETURN
    Revenue
```

During debugging, temporary control measures can validate each logical component separately.

The goal is to answer:

```text
Is StartDate correct?
Is EndDate correct?
Is ValidOrders correct?
Is the final population correct?
```

before changing the entire measure.

---

## 23. Debug Context Before Arithmetic

A useful troubleshooting sequence is:

```text
Unexpected KPI
      │
      ▼
Check Slicer
      │
      ▼
Check Page Filters
      │
      ▼
Check Visual Filters
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
Check Business Population
      │
      ▼
Check DAX Context
      │
      ▼
Only Then Check Arithmetic
```

This order matters.

Changing arithmetic before understanding context can hide the original problem and introduce new ones.

---

## 24. Avoid Changing Several Things at Once

When debugging DAX, changing:

- the relationship
- the date logic
- the status filter
- and the formula

at the same time makes it difficult to know what solved the problem.

A stronger approach is:

```text
One Question
     │
     ▼
One Hypothesis
     │
     ▼
One Targeted Test
     │
     ▼
Evaluate Result
     │
     ▼
Next Question
```

This creates a reproducible troubleshooting process.

---

## 25. Check Visual Context

A measure can return different values in:

- a card
- a table
- a matrix
- a chart

because each visual creates its own context.

For example, a table may evaluate a measure separately for every:

```text
Marketplace
Month
Customer
Product
```

while a card may evaluate only the total context.

Therefore, a measure should be tested in the type of visual where it will actually be used.

---

## 26. Totals Can Behave Differently from Rows

A Power BI total is not always the arithmetic sum of the visible rows.

The measure is often recalculated in the total context.

Conceptually:

```text
Row 1 → Measure evaluated in Row 1 context
Row 2 → Measure evaluated in Row 2 context
Row 3 → Measure evaluated in Row 3 context

Total → Measure evaluated again in Total context
```

This can surprise analysts when working with:

- ratios
- conditional logic
- iterators
- distinct counts
- context-dependent calculations

When a total looks unexpected, inspect the measure's evaluation context rather than assuming Power BI simply added the visible rows.

---

## 27. Distinct Count Requires Business Interpretation

`DISTINCTCOUNT` can correctly count unique values while still answering the wrong business question.

For example:

```DAX
DISTINCTCOUNT(Transactions[OrderID])
```

may count unique Order IDs in a settlement table.

But the result may still differ from operational orders because the settlement table represents financial transactions rather than the complete operational order population.

Therefore:

> **Correct DAX syntax does not guarantee correct KPI semantics.**

---

## 28. Blank Results Are Diagnostic Information

A blank measure should not immediately be replaced with zero.

A blank can indicate:

- no matching rows
- missing relationship
- wrong filter context
- no selected value
- no data for the period
- logic returning `BLANK()`

Replacing blanks with zero too early can hide useful debugging information.

During development, understand the blank first.

Format it later if the business requirement requires zero.

---

## 29. Performance Matters

Correct DAX should also be maintainable and performant.

Potential performance issues include:

- iterating very large tables unnecessarily
- repeated complex filters
- excessive bidirectional relationships
- unnecessary calculated columns
- repeated logic across measures
- large many-to-many operations

Optimization should happen after correctness has been established.

> **A fast wrong measure is still wrong.**

---

## 30. Document Complex Measures

Important measures should be understandable by another analyst.

Useful comments can explain:

```DAX
-- Reporting period comes from the central calendar
VAR StartDate =
    MIN(Calendar[Date])

-- Exclude cancelled marketplace orders
VAR ValidOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )
```

Comments should explain **why** logic exists, not simply repeat what the code already says.

---

## 31. DAX Validation Checklist

Before considering a measure validated, check:

### Business Logic

- What does the KPI mean?
- Which population should it include?
- Which records should it exclude?

### Date Context

- Which date controls the measure?
- Which table supplies the period?
- Does the source contain date-time values?

### Relationships

- Is the filter path active?
- Is the direction intentional?
- Is there an unwanted many-to-many relationship?

### Identifier Logic

- Are the compared keys compatible?
- Are duplicates understood?
- Is `TREATAS` appropriate?

### Financial Logic

- Gross or net?
- VAT included?
- Refunds included?
- Currency aligned?

### Visual Context

- Does the measure behave correctly in cards?
- Tables?
- Totals?
- Channel selections?

---

## 32. Reusable Debugging Workflow

A reusable Power BI debugging process is:

```text
1. Define expected business result
              ↓
2. Identify slicer and filter context
              ↓
3. Identify controlling date field
              ↓
4. Validate relationships
              ↓
5. Validate raw population
              ↓
6. Build simple control measure
              ↓
7. Add one business rule
              ↓
8. Validate again
              ↓
9. Add cross-table filtering
              ↓
10. Validate final measure
              ↓
11. Test in final visual
```

This is more reliable than repeatedly rewriting a complex formula until the number appears plausible.

---

## Final Takeaway

DAX debugging is not only about formulas.

A KPI result is produced by the interaction of:

```text
Source Data
     ↓
Data Model
     ↓
Relationships
     ↓
Filter Context
     ↓
Business Rules
     ↓
DAX
     ↓
Visual Context
     ↓
Result
```

When a number looks wrong, any one of these layers may be responsible.

The most important lesson from this project was therefore:

> **Do not immediately rewrite a measure because the result looks wrong. First determine which data, relationships, dates, and filters the measure is actually seeing.**
