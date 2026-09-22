# DAX Revenue Measures

This file contains reusable DAX patterns for revenue analysis and reconciliation in Power BI.

The examples use generic table and column names so that the patterns can be adapted to other projects.

> **Important:** Revenue should only be compared after the financial definition, order population, reporting period, marketplace, currency, VAT treatment, and refund logic have been aligned.

---

## 1. Basic Gross Sales

A simple gross-sales measure can be:

```DAX
Gross Sales =
SUM(
    MonthlyReport[GrossSales]
)
```

This is only meaningful if `GrossSales` has a documented source definition.

Before using the measure, confirm whether the value includes:

- VAT
- shipping
- discounts
- refunds
- marketplace taxes

---

## 2. Sales From Multiple Components

Some reporting sources split sales into separate components.

For example:

```DAX
Reported Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
```

Before adding components, verify that they do not overlap.

Do not automatically sum every field containing the word `Sales`.

---

## 3. Source-Specific Net Sales Example

Some sources may provide gross-like sales components together with VAT stored as a negative value.

A source-specific measure could then be:

```DAX
Net Sales =
SUM(MonthlyReport[OrganicSales])
    + SUM(MonthlyReport[PPCSales])
    + SUM(MonthlyReport[VAT])
```

This pattern is valid only when the source semantics have been verified.

For example:

```text
Sales = VAT-inclusive amount
VAT   = negative tax component
```

The formula should not be treated as a universal accounting rule.

---

## 4. ERP Net Revenue

If the ERP contains validated net revenue at order-line level:

```DAX
ERP Net Revenue =
SUM(
    ERPOrderLines[NetRevenue]
)
```

Before using the result, confirm the table grain.

If one order contains multiple lines, summing order-line revenue can be correct while counting rows as orders would not be.

---

## 5. Revenue for Valid Orders

Revenue should normally use the same intended order population as the related order KPI.

A reusable pattern is:

```DAX
Revenue for Valid Orders =
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

Conceptually:

```text
Validated Marketplace Orders
           │
           ▼
        TREATAS
           │
           ▼
ERP External Order IDs
           │
           ▼
ERP Order-Line Revenue
```

This helps keep the order and revenue populations aligned.

---

## 6. Revenue Controlled by a Calendar Period

When the reporting period comes from a central calendar:

```DAX
Revenue by Selected Period =
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])

VAR ValidOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled",
        FILTER(
            ALL(AmazonOrders[PurchaseDate]),
            AmazonOrders[PurchaseDate] >= StartDate
                &&
            AmazonOrders[PurchaseDate] < EndDate + 1
        )
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

This explicitly defines the order population before revenue is calculated.

---

## 7. Revenue Controlled by a Source-Specific Period

A source-analysis page may receive its period from another table.

For example:

```DAX
Revenue by Source Period =
VAR StartDate =
    MIN(MonthlyReport[DateFrom])

VAR EndDate =
    MAX(MonthlyReport[DateTo])

VAR ValidOrders =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled",
        FILTER(
            ALL(AmazonOrders[PurchaseDate]),
            AmazonOrders[PurchaseDate] >= StartDate
                &&
            AmazonOrders[PurchaseDate] < EndDate + 1
        )
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

The financial logic can remain unchanged while the reporting-period source changes.

---

## 8. Why the Order Population Matters

Consider:

```text
Order KPI
→ excludes cancelled orders

Revenue KPI
→ includes all ERP orders
```

The two KPIs are then based on different populations.

A better design is:

```text
Validated Order Population
          │
          ├──► Order Count
          │
          └──► Revenue
```

This makes the dashboard easier to explain and validate.

---

## 9. Marketplace-Specific Revenue

If a dedicated marketplace field is available:

```DAX
Marketplace Revenue =
CALCULATE(
    [Revenue for Valid Orders],
    AmazonOrders[SalesChannel] = "Marketplace-DE"
)
```

Marketplace filtering should use the appropriate marketplace or sales-channel field.

Shipping country should not automatically be used as a marketplace substitute.

---

## 10. Revenue Difference

When two equivalent revenue concepts have been aligned:

```DAX
Revenue Difference =
[ERP Net Revenue]
    - [Marketplace Net Revenue]
```

This measure quantifies the source difference.

It does not by itself explain the cause.

---

## 11. Absolute Revenue Difference

For monitoring:

```DAX
Absolute Revenue Difference =
ABS(
    [Revenue Difference]
)
```

This can be useful when the magnitude of the discrepancy matters more than its direction.

---

## 12. Revenue Difference Percentage

A relative difference can be calculated as:

```DAX
Revenue Difference % =
DIVIDE(
    [Revenue Difference],
    [Marketplace Net Revenue]
)
```

The denominator should always be documented.

For example:

```text
Difference relative to marketplace net revenue
```

Changing the denominator changes the business interpretation.

---

## 13. Gross-to-Net Difference

If both validated gross and net values exist:

```DAX
Gross to Net Difference =
[Gross Sales]
    - [Net Sales]
```

This difference may contain tax and potentially other source-specific components.

Do not automatically label the entire difference as VAT unless the source definition proves that interpretation.

---

## 14. VAT Control

If VAT is stored separately:

```DAX
VAT Amount =
SUM(
    MonthlyReport[VAT]
)
```

The sign convention must be understood.

For example:

```text
+ VAT
```

and:

```text
- VAT
```

can require different formulas.

Never infer the accounting treatment only from the column name.

---

## 15. Refund Control

If the source contains a dedicated refund amount:

```DAX
Refund Amount =
SUM(
    MonthlyReport[RefundAmount]
)
```

Before combining refunds with revenue, determine:

- whether the refund amount is positive or negative
- which date controls the refund
- whether VAT adjustments are included
- whether the main sales metric already reflects refunds

---

## 16. Shipping Revenue

If customer shipping charges are stored separately:

```DAX
Shipping Revenue =
SUM(
    Settlement[ShippingCredit]
)
```

Do not confuse:

```text
Customer Shipping Revenue
```

with:

```text
Carrier Shipping Cost
```

or:

```text
Fulfilment Fee
```

These represent different financial concepts.

---

## 17. Marketplace Fees

A separate measure can make fees visible:

```DAX
Marketplace Fees =
SUM(
    Settlement[MarketplaceFees]
)
```

Fees should not automatically be subtracted from sales when the business question is:

> What was the sales revenue?

Fees belong to a different financial layer unless the KPI definition explicitly includes them.

---

## 18. Settlement Total

A settlement file may contain a final transaction amount:

```DAX
Settlement Total =
SUM(
    Settlement[Total]
)
```

This should not automatically be compared with ERP sales revenue.

Settlement totals can include:

- sales
- refunds
- fees
- reimbursements
- adjustments
- previous-period transactions

Therefore:

> **Settlement total is not automatically revenue.**

---

## 19. Payout Is Not Revenue

A payout measure might be:

```DAX
Estimated Payout =
SUM(
    MonthlyReport[EstimatedPayout]
)
```

This can be useful for cash-flow analysis.

It should not automatically replace a sales-revenue KPI.

Conceptually:

```text
Revenue
   ↓
Financial Adjustments
   ↓
Fees / Refunds / Other Effects
   ↓
Payout
```

These are different analytical stages.

---

## 20. Order-Level Amount Difference

Where record-level reconciliation exists, an amount difference can be calculated:

```DAX
Order Amount Difference =
[Source A Amount]
    - [Source B Amount]
```

This is more informative than relying only on aggregate totals.

---

## 21. Amount Match With Tolerance

Small differences may result from rounding.

A controlled tolerance example is:

```DAX
Amount Match =
IF(
    ABS(
        [Source A Amount]
            - [Source B Amount]
    ) <= 0.01,
    "Matched",
    "Investigate"
)
```

The tolerance should reflect the actual business and currency requirements.

It should not be chosen merely to hide unexplained differences.

---

## 22. Amount Match Rate

If a reconciliation table contains a match result:

```DAX
Matched Amount Records =
CALCULATE(
    COUNTROWS(Reconciliation),
    Reconciliation[AmountMatch] = "Matched"
)
```

and:

```DAX
Amount Match Rate =
DIVIDE(
    [Matched Amount Records],
    COUNTROWS(Reconciliation)
)
```

This provides a more detailed quality indicator than comparing only grand totals.

---

## 23. Revenue by Currency

For international data, revenue should not be aggregated blindly across currencies.

A simple diagnostic measure is:

```DAX
Revenue by Currency =
SUM(
    Transactions[Revenue]
)
```

used together with:

```text
Transactions[Currency]
```

in a table or matrix.

Before creating a combined reporting-currency KPI, define:

- exchange-rate source
- exchange-rate date
- conversion method
- rounding method

---

## 24. Revenue by Marketplace

A diagnostic measure can be:

```DAX
Revenue by Marketplace =
SUM(
    Transactions[Revenue]
)
```

displayed with:

```text
Transactions[Marketplace]
```

This helps identify whether an apparent total discrepancy is actually caused by different marketplace scope.

---

## 25. Management Revenue With Channel Logic

A management report may require channel-specific calculations.

A simplified pattern is:

```DAX
Management Revenue =
VAR SelectedChannel =
    SELECTEDVALUE(
        Channel[Channel]
    )

RETURN
SWITCH(
    TRUE(),

    SelectedChannel = "Amazon",
        [Amazon Net Revenue],

    SelectedChannel = "eBay",
        [eBay Net Revenue],

    [Default Net Revenue]
)
```

Each branch should use a validated source-specific definition.

A single KPI label does not mean every source calculates revenue in the same way.

---

## 26. Avoid Double Counting

Suppose a source contains:

```text
Organic Sales
PPC Sales
Sponsored Sales
```

Before writing:

```DAX
Total Sales =
[Organic Sales]
    + [PPC Sales]
    + [Sponsored Sales]
```

verify whether sponsored sales are already included in PPC sales.

Otherwise, the calculation can double count revenue while still looking mathematically valid.

---

## 27. Avoid Mixing Revenue and Fees

This type of calculation may answer a profit or payout-related question:

```DAX
Sales
+ Refunds
+ Fees
+ Shipping
```

but it should not automatically be called:

```text
Revenue
```

Financial KPIs should be named according to what they actually represent.

---

## 28. Avoid Correction Factors Without a Business Rule

Do not use:

```DAX
Corrected Revenue =
[Revenue] * 0.98
```

simply because another source is approximately 2% lower.

A correction factor can hide:

- wrong VAT logic
- different order populations
- marketplace differences
- missing records
- timing differences
- source-system behavior

Only use correction logic when a documented business rule justifies it.

---

## 29. Aggregate Equality Is Not Enough

Suppose:

```text
Source A Revenue = 100,000
Source B Revenue = 100,000
```

This does not prove that the underlying orders match.

For example:

```text
Order A Difference   +500
Order B Difference   -500
-------------------------
Total Difference        0
```

Therefore, high-value reconciliations should combine:

```text
Aggregate Validation
        +
Record-Level Validation
```

---

## 30. Useful Revenue Control Measures

During development, useful controls include:

```text
Gross Sales
Net Sales
VAT
Refunds
ERP Net Revenue
Revenue for Valid Orders
Revenue by Marketplace
Revenue by Currency
Revenue Difference
Revenue Difference %
```

These measures make it easier to isolate the layer where a discrepancy appears.

---

## 31. Recommended Validation Sequence

A reliable revenue reconciliation can follow:

```text
Define Financial Metric
        ↓
Validate Order Population
        ↓
Align Reporting Period
        ↓
Align Marketplace
        ↓
Align Currency
        ↓
Validate Gross / Net
        ↓
Validate VAT
        ↓
Validate Refunds
        ↓
Validate Shipping Components
        ↓
Separate Fees / Payout
        ↓
Compare Order-Level Amounts
        ↓
Quantify Residual Difference
```

---

## 32. Revenue Validation Checklist

Before treating a revenue measure as validated, confirm:

- [ ] Financial metric clearly defined
- [ ] Gross/net treatment documented
- [ ] VAT treatment understood
- [ ] Order population aligned
- [ ] Cancelled-order treatment aligned
- [ ] Reporting period aligned
- [ ] Marketplace scope aligned
- [ ] Currency aligned
- [ ] Refund treatment understood
- [ ] Shipping components understood
- [ ] Fees separated where appropriate
- [ ] Payout not confused with revenue
- [ ] Filter context validated
- [ ] Aggregate totals checked
- [ ] Record-level samples checked
- [ ] Remaining difference quantified
- [ ] Unresolved differences documented

---

## Final Principle

A revenue measure is not trustworthy simply because the arithmetic is correct.

Conceptually:

```text
Business Definition
        ↓
Order Population
        ↓
Reporting Period
        ↓
Marketplace
        ↓
Currency
        ↓
Gross / Net
        ↓
VAT
        ↓
Refunds / Adjustments
        ↓
Filter Context
        ↓
Revenue KPI
```

The most important question is not:

> **Which monetary column should I sum?**

It is:

> **What financial concept does this number represent, and is it equivalent to the number I am comparing it with?**

> **A trustworthy revenue reconciliation explains the meaning of the numbers before it explains the difference between them.**
