# DAX Order Measures

This file contains reusable DAX patterns for order analysis and reconciliation in Power BI.

The examples use generic table and column names so that the patterns can be adapted to other projects.

> **Important:** A technically correct order count is only meaningful when the business definition, reporting period, marketplace, status rules, and table grain are clearly defined.

---

## 1. Raw Distinct Orders

A basic control measure for counting unique Order IDs:

```DAX
Raw Orders =
DISTINCTCOUNT(
    AmazonOrders[OrderID]
)
```

This is useful as an initial validation measure.

It does not apply business rules such as:

- cancellation exclusion
- marketplace filtering
- date alignment
- identifier normalization

Therefore, it should not automatically be treated as the final business KPI.

---

## 2. Valid Non-Cancelled Orders

A common business rule is to exclude cancelled orders:

```DAX
Valid Orders =
CALCULATE(
    DISTINCTCOUNT(
        AmazonOrders[OrderID]
    ),
    AmazonOrders[OrderStatus] <> "Canceled"
)
```

This measure answers a more specific question:

> How many distinct orders remain after cancelled orders are excluded?

The exact status logic should always be validated against the source system.

---

## 3. Cancelled Orders

A useful control measure is:

```DAX
Canceled Orders =
CALCULATE(
    DISTINCTCOUNT(
        AmazonOrders[OrderID]
    ),
    AmazonOrders[OrderStatus] = "Canceled"
)
```

This helps explain the difference between raw and cleaned order populations.

Conceptually:

```text
Raw Orders
    │
    ├── Valid Orders
    │
    └── Canceled Orders
```

---

## 4. Orders Controlled by a Calendar Period

When the report period comes from a central calendar table, the selected range can be transferred explicitly to the order source.

```DAX
Orders by Selected Period =
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])

RETURN
CALCULATE(
    DISTINCTCOUNT(
        AmazonOrders[OrderID]
    ),
    AmazonOrders[OrderStatus] <> "Canceled",
    FILTER(
        ALL(AmazonOrders[PurchaseDate]),
        AmazonOrders[PurchaseDate] >= StartDate
            &&
        AmazonOrders[PurchaseDate] < EndDate + 1
    )
)
```

This pattern is useful when the order table does not automatically receive the intended date context.

---

## 5. Why `< EndDate + 1` Is Used

If `PurchaseDate` contains date-time values, records on the final selected day may contain times such as:

```text
2026-02-28 08:15:00
2026-02-28 17:42:00
2026-02-28 23:59:59
```

Using:

```DAX
AmazonOrders[PurchaseDate] < EndDate + 1
```

includes the entire final day.

Conceptually:

```text
StartDate <= PurchaseDate < Day After EndDate
```

---

## 6. Orders Controlled by a Source-Specific Period

A source-analysis page may use a monthly report table instead of the central calendar.

For example:

```DAX
Orders by Source Period =
VAR StartDate =
    MIN(MonthlyReport[DateFrom])

VAR EndDate =
    MAX(MonthlyReport[DateTo])

RETURN
CALCULATE(
    DISTINCTCOUNT(
        AmazonOrders[OrderID]
    ),
    AmazonOrders[OrderStatus] <> "Canceled",
    FILTER(
        ALL(AmazonOrders[PurchaseDate]),
        AmazonOrders[PurchaseDate] >= StartDate
            &&
        AmazonOrders[PurchaseDate] < EndDate + 1
    )
)
```

The business logic is similar to the previous measure.

The important difference is the source of the reporting period.

---

## 7. Marketplace-Specific Orders

If the analysis requires a specific marketplace:

```DAX
Marketplace Orders =
CALCULATE(
    [Valid Orders],
    AmazonOrders[SalesChannel] = "Marketplace-DE"
)
```

Use the dedicated marketplace or sales-channel field whenever possible.

Do not automatically substitute shipping country for marketplace.

---

## 8. Orders by Selected Marketplace

For reports where marketplace is selected dynamically:

```DAX
Orders by Selected Marketplace =
VAR SelectedMarketplace =
    SELECTEDVALUE(
        Marketplace[Marketplace]
    )

RETURN
CALCULATE(
    [Valid Orders],
    AmazonOrders[SalesChannel] = SelectedMarketplace
)
```

This pattern assumes that one marketplace is selected.

If multiple selections are allowed, the model should use relationship-based filtering or another appropriate multi-value filtering strategy.

---

## 9. Raw vs. Valid Order Difference

A useful data-quality measure is:

```DAX
Excluded Orders =
[Raw Orders]
    - [Valid Orders]
```

This quantifies the effect of the business-rule exclusions.

It does not automatically explain why every excluded record was removed.

---

## 10. Source Order Difference

When comparing two source systems:

```DAX
Order Difference =
[Source A Orders]
    - [Source B Orders]
```

This should be interpreted as:

> Difference between the two source counts under the current definitions.

It should not automatically be interpreted as:

> Missing orders.

The cause must be investigated separately.

---

## 11. Absolute Order Difference

For monitoring purposes, the direction of the difference may be less important than its size.

```DAX
Absolute Order Difference =
ABS(
    [Order Difference]
)
```

This can be useful for data-quality monitoring.

---

## 12. Order Difference Percentage

A relative difference can provide additional context:

```DAX
Order Difference % =
DIVIDE(
    [Order Difference],
    [Source A Orders]
)
```

The denominator should be documented because changing the reference source changes the interpretation.

---

## 13. Orders With Blank IDs

A simple data-quality control:

```DAX
Orders With Blank ID =
CALCULATE(
    COUNTROWS(AmazonOrders),
    ISBLANK(AmazonOrders[OrderID])
)
```

Records without usable identifiers may not participate reliably in reconciliation.

They should therefore remain visible as a data-quality issue.

---

## 14. Distinct Status Count

A diagnostic measure can help identify unexpected status behavior:

```DAX
Distinct Order Statuses =
DISTINCTCOUNT(
    AmazonOrders[OrderStatus]
)
```

The actual status values should also be inspected in a table or matrix.

A new status value can affect existing KPI logic even when the DAX measure still executes successfully.

---

## 15. Order Population as a Reusable Table Expression

For more complex calculations, a validated order population can be defined inside a measure.

```DAX
VAR ValidOrderIDs =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )
```

This population can then be reused inside the same measure for controlled cross-table filtering.

---

## 16. Filtering Another Table With Valid Order IDs

If operational Order IDs need to filter an ERP table:

```DAX
Orders Found in ERP =
VAR ValidOrderIDs =
    CALCULATETABLE(
        VALUES(AmazonOrders[OrderID]),
        AmazonOrders[OrderStatus] <> "Canceled"
    )

RETURN
CALCULATE(
    DISTINCTCOUNT(
        ERPOrders[ExternalOrderID]
    ),
    TREATAS(
        ValidOrderIDs,
        ERPOrders[ExternalOrderID]
    )
)
```

`TREATAS` transfers the validated identifier population to another table.

Before using this pattern, verify that:

- both columns represent compatible business identifiers
- blanks are understood
- duplicates are understood
- marketplace scope is aligned
- formatting is compatible

---

## 17. Orders Not Found in the Comparison Population

A high-level diagnostic measure can be:

```DAX
Unmatched Order Count =
[Valid Orders]
    - [Orders Found in ERP]
```

However, this is only a count-level control.

A proper reconciliation process should create a record-level comparison table before individual orders are classified as truly unmatched.

---

## 18. Status Distribution

A table visual can use:

```text
OrderStatus
```

together with:

```DAX
Orders by Status =
DISTINCTCOUNT(
    AmazonOrders[OrderID]
)
```

This allows analysts to inspect the source population before exclusions are applied.

Example output:

```text
Status       Orders
-------------------
Shipped      ...
Canceled     ...
Pending      ...
Other        ...
```

---

## 19. Control Measure Strategy

A reliable order KPI should be built incrementally.

Recommended sequence:

```text
Raw Distinct Orders
        ↓
Reporting Period
        ↓
Status Rules
        ↓
Marketplace
        ↓
Identifier Validation
        ↓
Cross-System Matching
        ↓
Final KPI
```

At every stage, compare the result with the previous stage.

This makes unexpected population changes easier to explain.

---

## 20. Example Management Order KPI

A management KPI may need channel-specific logic.

A simplified example:

```DAX
Management Orders =
VAR SelectedChannel =
    SELECTEDVALUE(
        Channel[Channel]
    )

RETURN
SWITCH(
    TRUE(),

    SelectedChannel = "Amazon",
        [Orders by Selected Period],

    SelectedChannel = "eBay",
        [eBay Valid Orders],

    [Default Orders]
)
```

Each branch should have its own validated business definition.

The fact that several measures are displayed under the label **Orders** does not mean that their source systems use identical logic.

---

## 21. Do Not Normalize Identifiers Blindly

It can be tempting to transform identifiers such as:

```text
ORDER-123_1
ORDER-123_2
```

into:

```text
ORDER-123
```

and then use:

```DAX
DISTINCTCOUNT(
    Orders[NormalizedOrderID]
)
```

This should only become production logic after the suffix semantics are understood.

Until then, normalization is better treated as an analytical matching technique.

---

## 22. Do Not Use COUNTROWS Without Understanding Grain

This measure:

```DAX
Order Rows =
COUNTROWS(AmazonOrders)
```

counts rows.

It does not necessarily count business orders.

If one order can appear several times, the result may be much larger than the actual order count.

Always determine whether the table is:

```text
Order Level
Order-Line Level
Transaction Level
```

before choosing the aggregation.

---

## 23. Do Not Treat a Matching Total as Proof

Suppose:

```text
Source A Orders = 1,000
Source B Orders = 1,000
```

This does not prove that the same 1,000 orders exist in both systems.

For example:

```text
10 records missing from Source B
10 different extra records in Source B
```

can still produce identical totals.

Therefore, important reconciliations should include record-level identifier matching.

---

## 24. Recommended Validation Measures

Useful temporary controls include:

```text
Raw Orders
Valid Orders
Canceled Orders
Orders by Marketplace
Orders by Period
Orders With Blank ID
Orders Found in Comparison Source
Source Difference
```

These controls can later be hidden from report consumers while remaining available for maintenance and troubleshooting.

---

## 25. Recommended Naming

Measure names should describe the business meaning clearly.

Prefer:

```text
Valid Orders
Canceled Orders
Amazon Orders
Order Difference
```

over names such as:

```text
Measure1
Test2
Final_New
Final_New_Test
```

Temporary test measures are useful during development, but production measures should eventually receive clear names.

---

## 26. Validation Checklist

Before treating an order measure as validated, confirm:

- [ ] Business definition documented
- [ ] Table grain understood
- [ ] Business identifier validated
- [ ] Blank IDs quantified
- [ ] Duplicate behavior understood
- [ ] Reporting date defined
- [ ] Date-time behavior checked
- [ ] Status rules validated
- [ ] Marketplace scope validated
- [ ] Filter context validated
- [ ] Cross-table filtering validated
- [ ] Aggregate result checked
- [ ] Record-level samples checked
- [ ] Remaining differences documented

---

## Final Principle

A reliable order measure is not simply:

```DAX
DISTINCTCOUNT(OrderID)
```

It is the result of a clearly defined business population.

Conceptually:

```text
Source Records
      ↓
Correct Grain
      ↓
Correct Period
      ↓
Correct Status
      ↓
Correct Marketplace
      ↓
Valid Identifiers
      ↓
Validated Filter Context
      ↓
Order KPI
```

> **Count business objects only after defining which business objects belong in the count.**
