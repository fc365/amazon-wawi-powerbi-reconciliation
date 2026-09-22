# DAX Reconciliation Measures

This file contains reusable DAX patterns for reconciliation, source comparison, residual analysis, and data-quality monitoring in Power BI.

The examples use generic table and column names so they can be adapted to other projects.

> **Important:** A reconciliation measure should quantify a difference before attempting to explain it. A numerical difference alone does not prove that records are missing or incorrect.

---

## 1. Source Order Difference

A basic comparison between two order KPIs:

```DAX
Order Difference =
[Source A Orders]
    - [Source B Orders]
```

This answers:

> How large is the numerical difference between the two source populations?

It does not answer:

> Why does the difference exist?

---

## 2. Absolute Order Difference

If only the magnitude matters:

```DAX
Absolute Order Difference =
ABS(
    [Order Difference]
)
```

This can be useful for monitoring and alerting.

---

## 3. Order Difference Percentage

A relative comparison can be calculated as:

```DAX
Order Difference % =
DIVIDE(
    [Order Difference],
    [Source A Orders]
)
```

The denominator should always be documented.

For example:

```text
Difference relative to Source A
```

---

## 4. Source Revenue Difference

After financial definitions have been aligned:

```DAX
Revenue Difference =
[Source A Net Revenue]
    - [Source B Net Revenue]
```

This measure should only be used when the compared values represent sufficiently equivalent financial concepts.

---

## 5. Absolute Revenue Difference

```DAX
Absolute Revenue Difference =
ABS(
    [Revenue Difference]
)
```

This is useful for monitoring the size of a discrepancy independently of direction.

---

## 6. Revenue Difference Percentage

```DAX
Revenue Difference % =
DIVIDE(
    [Revenue Difference],
    [Source B Net Revenue]
)
```

Again, document the reference source because the denominator determines the interpretation.

---

## 7. Match Status

A reconciliation table can classify records based on whether the business identifier exists in both sources.

For example:

```DAX
Match Status =
IF(
    NOT ISBLANK(Reconciliation[SourceAOrderID])
        &&
    NOT ISBLANK(Reconciliation[SourceBOrderID]),
    "Matched",
    "Unmatched"
)
```

For larger models, this classification may be better implemented during Power Query or reconciliation-table construction rather than as a calculated column in DAX.

---

## 8. More Detailed Match Status

A more informative classification can distinguish the side on which the record exists:

```DAX
Match Status Detailed =
SWITCH(
    TRUE(),

    NOT ISBLANK(Reconciliation[SourceAOrderID])
        &&
    NOT ISBLANK(Reconciliation[SourceBOrderID]),
        "Matched",

    NOT ISBLANK(Reconciliation[SourceAOrderID])
        &&
    ISBLANK(Reconciliation[SourceBOrderID]),
        "Source A Only",

    ISBLANK(Reconciliation[SourceAOrderID])
        &&
    NOT ISBLANK(Reconciliation[SourceBOrderID]),
        "Source B Only",

    "Unknown"
)
```

This is more useful than labeling every difference simply as missing.

---

## 9. Matched Orders

If a reconciliation table already contains a match classification:

```DAX
Matched Orders =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[MatchStatus] = "Matched"
)
```

This measures confirmed identifier matches within the reconciliation population.

---

## 10. Unmatched Orders

```DAX
Unmatched Orders =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[MatchStatus] <> "Matched"
)
```

The term `Unmatched` should mean:

> Not matched under the current reconciliation logic.

It should not automatically mean:

> Missing from the business process.

---

## 11. Source A Only

```DAX
Source A Only Orders =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[MatchStatus] = "Source A Only"
)
```

---

## 12. Source B Only

```DAX
Source B Only Orders =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[MatchStatus] = "Source B Only"
)
```

Together, these measures show the direction of identifier-level differences.

---

## 13. Match Rate

```DAX
Order Match Rate =
DIVIDE(
    [Matched Orders],
    [Matched Orders] + [Unmatched Orders]
)
```

A high match rate is useful evidence.

It does not automatically prove that all matched records also have matching financial values.

---

## 14. Amount Difference Per Record

For records available in both sources:

```DAX
Amount Difference =
Reconciliation[SourceAAmount]
    - Reconciliation[SourceBAmount]
```

This can be stored as a calculated column if the reconciliation table is intentionally modeled at the comparison grain.

---

## 15. Amount Match With Tolerance

Small financial differences may be caused by rounding.

A controlled example:

```DAX
Amount Match =
IF(
    ABS(
        Reconciliation[SourceAAmount]
            - Reconciliation[SourceBAmount]
    ) <= 0.01,
    "Matched",
    "Investigate"
)
```

The tolerance must be justified by the business and currency context.

It should not be increased simply to make more records appear matched.

---

## 16. Matched Amount Records

```DAX
Matched Amount Records =
CALCULATE(
    COUNTROWS(Reconciliation),
    Reconciliation[AmountMatch] = "Matched"
)
```

---

## 17. Amount Mismatch Records

```DAX
Amount Mismatch Records =
CALCULATE(
    COUNTROWS(Reconciliation),
    Reconciliation[AmountMatch] = "Investigate"
)
```

---

## 18. Amount Match Rate

```DAX
Amount Match Rate =
DIVIDE(
    [Matched Amount Records],
    [Matched Amount Records]
        + [Amount Mismatch Records]
)
```

This gives a record-level financial quality indicator.

---

## 19. Total Residual Amount

```DAX
Residual Amount =
SUMX(
    FILTER(
        Reconciliation,
        Reconciliation[AmountMatch] = "Investigate"
    ),
    Reconciliation[SourceAAmount]
        - Reconciliation[SourceBAmount]
)
```

This quantifies the remaining amount difference among records requiring investigation.

---

## 20. Absolute Residual Amount

Positive and negative differences can cancel each other out.

Therefore, an additional control can be:

```DAX
Absolute Residual Amount =
SUMX(
    FILTER(
        Reconciliation,
        Reconciliation[AmountMatch] = "Investigate"
    ),
    ABS(
        Reconciliation[SourceAAmount]
            - Reconciliation[SourceBAmount]
    )
)
```

This helps reveal record-level discrepancies that could be hidden by a near-zero net difference.

---

## 21. Reason Codes

A reconciliation process becomes more useful when unmatched records are classified by reason.

Possible values include:

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

Reason codes should be based on evidence.

They should not be assigned from assumptions merely to eliminate the unresolved category.

---

## 22. Unresolved Orders

If a reason-code field exists:

```DAX
Unresolved Orders =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[ReasonCode] = "UNRESOLVED"
)
```

Keeping unresolved cases visible is an important part of transparent reporting.

---

## 23. Unresolved Amount

```DAX
Unresolved Amount =
CALCULATE(
    SUM(
        Reconciliation[SourceAAmount]
    ),
    Reconciliation[ReasonCode] = "UNRESOLVED"
)
```

The correct amount field depends on the purpose of the reconciliation.

The chosen source should be documented.

---

## 24. Canceled Reconciliation Cases

```DAX
Canceled Cases =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[ReasonCode] = "CANCELED"
)
```

This can help explain why an operational source and a cleaned business KPI differ.

---

## 25. Later Settlement Cases

```DAX
Settled Later Cases =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[ReasonCode] = "SETTLED_LATER"
)
```

This separates timing differences from genuinely unresolved cases.

---

## 26. Identifier Variation Cases

```DAX
Identifier Variation Cases =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[ReasonCode] = "IDENTIFIER_VARIATION"
)
```

Identifier normalization should only be classified as confirmed when the matching rule has been validated.

---

## 27. Marketplace Difference Cases

```DAX
Marketplace Difference Cases =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[ReasonCode] = "MARKETPLACE_DIFFERENCE"
)
```

This helps distinguish marketplace-scope problems from identifier or financial problems.

---

## 28. Explained Cases

If the reconciliation table contains evidence-based reason codes:

```DAX
Explained Cases =
CALCULATE(
    DISTINCTCOUNT(
        Reconciliation[BusinessOrderID]
    ),
    Reconciliation[ReasonCode] <> "UNRESOLVED"
)
```

Depending on the model, `MATCHED` may need to be treated separately from discrepancy explanations.

The exact definition should be documented.

---

## 29. Explained Difference Rate

For discrepancy records only, a useful KPI can be:

```DAX
Explained Difference Rate =
DIVIDE(
    [Explained Difference Cases],
    [Total Difference Cases]
)
```

This requires clearly defined supporting measures.

The KPI should answer:

> What share of investigated discrepancy cases has an evidence-supported explanation?

It should not include records that were never part of the discrepancy population.

---

## 30. Reconciliation Status KPI

A high-level monitoring measure can classify reconciliation health.

For example:

```DAX
Reconciliation Status =
SWITCH(
    TRUE(),

    [Unresolved Orders] = 0,
        "No Unresolved Cases",

    [Unresolved Orders] <= 5,
        "Review",

    "Investigation Required"
)
```

Thresholds in production should be agreed with business stakeholders.

They should not be selected arbitrarily.

---

## 31. Data Quality Status

A more general quality indicator can combine several controls:

```DAX
Data Quality Status =
SWITCH(
    TRUE(),

    [Orders With Blank ID] > 0,
        "Critical",

    [Unresolved Orders] > 0,
        "Review",

    "OK"
)
```

This is only an example.

Real production severity rules should reflect business impact.

---

## 32. Selected Period Reconciliation

A reconciliation measure should normally respect the same reporting period across sources.

A generic pattern is:

```DAX
Source A Orders Selected Period =
VAR StartDate =
    MIN(Calendar[Date])

VAR EndDate =
    MAX(Calendar[Date])

RETURN
CALCULATE(
    DISTINCTCOUNT(SourceA[OrderID]),
    FILTER(
        ALL(SourceA[OrderDate]),
        SourceA[OrderDate] >= StartDate
            &&
        SourceA[OrderDate] < EndDate + 1
    )
)
```

The equivalent Source B measure should use the same intended business period.

---

## 33. Reconciliation by Marketplace

A reconciliation page can expose differences by:

```text
Marketplace
```

using measures such as:

```DAX
Order Difference =
[Source A Orders]
    - [Source B Orders]
```

When placed in a matrix with marketplace, the measure is evaluated separately for each marketplace context.

This can reveal scope differences hidden by the overall total.

---

## 34. Reconciliation by Month

The same measures can be analyzed by month:

```text
Month
Source A Orders
Source B Orders
Order Difference
Source A Revenue
Source B Revenue
Revenue Difference
```

This helps determine whether a discrepancy is:

- isolated
- recurring
- seasonal
- associated with a source change

---

## 35. Reconciliation Trend

A useful report can track:

```text
Order Difference %
Revenue Difference %
Unresolved Orders
Amount Match Rate
```

over time.

The goal is not necessarily to force every metric to zero.

The goal is to make changes in reconciliation quality visible.

---

## 36. Control Totals

Before trusting detailed reconciliation logic, compare simple totals.

Examples:

```DAX
Source A Raw Orders =
DISTINCTCOUNT(SourceA[OrderID])
```

```DAX
Source B Raw Orders =
DISTINCTCOUNT(SourceB[OrderID])
```

```DAX
Source A Raw Revenue =
SUM(SourceA[Revenue])
```

```DAX
Source B Raw Revenue =
SUM(SourceB[Revenue])
```

These controls help establish the starting population before business rules are added.

---

## 37. Build Reconciliation Incrementally

A strong reconciliation process can follow:

```text
Raw Source A
      +
Raw Source B
      ↓
Align Period
      ↓
Align Grain
      ↓
Align Status
      ↓
Align Marketplace
      ↓
Exact Identifier Match
      ↓
Timing Investigation
      ↓
Identifier Investigation
      ↓
Amount Comparison
      ↓
Reason Classification
      ↓
Residual Cases
```

Each stage should reduce uncertainty.

---

## 38. Do Not Use Aggregate Differences Alone

This measure:

```DAX
Order Difference =
[Source A Orders] - [Source B Orders]
```

is useful.

But it cannot identify which records differ.

Likewise:

```DAX
Revenue Difference =
[Source A Revenue] - [Source B Revenue]
```

cannot reveal whether positive and negative record-level differences cancel each other.

A robust reconciliation combines:

```text
Aggregate KPIs
      +
Record-Level Matching
      +
Reason Classification
```

---

## 39. Do Not Call Every Unmatched Record Missing

Use terminology carefully.

Prefer:

```text
Unmatched
Source A Only
Source B Only
Residual
Unresolved
```

until the evidence proves a stronger classification.

This improves analytical accuracy and stakeholder communication.

---

## 40. Do Not Force Zero Difference

A zero difference can look attractive, but it is not the objective of reconciliation.

Artificial methods such as:

- unexplained correction factors
- aggressive identifier normalization
- removing residual records
- increasing amount tolerance

can make the dashboard look cleaner while reducing trustworthiness.

The objective is:

> **Explain the difference as far as the evidence allows.**

---

## 41. Recommended Reconciliation Dashboard

A dedicated reconciliation page can contain:

```text
Source A Orders
Source B Orders
Order Difference
Order Difference %

Source A Revenue
Source B Revenue
Revenue Difference
Revenue Difference %

Matched Orders
Unmatched Orders
Unresolved Orders
Amount Match Rate
```

Additional tables can show:

```text
Order ID
Marketplace
Status
Source A Amount
Source B Amount
Difference
Reason Code
Evidence Status
```

This keeps technical diagnostics separate from the management dashboard.

---

## 42. Evidence Status

A reconciliation table can optionally contain an evidence classification such as:

```text
CONFIRMED
EMPIRICALLY_SUPPORTED
HYPOTHESIS
UNRESOLVED
```

A measure can then quantify confirmed cases:

```DAX
Confirmed Cases =
CALCULATE(
    COUNTROWS(Reconciliation),
    Reconciliation[EvidenceStatus] = "CONFIRMED"
)
```

This prevents hypotheses from being presented as proven explanations.

---

## 43. Recommended Validation Checklist

Before treating a reconciliation KPI as reliable, confirm:

- [ ] Both source definitions documented
- [ ] Table grain understood
- [ ] Reporting period aligned
- [ ] Business dates aligned
- [ ] Status rules aligned
- [ ] Marketplace scope aligned
- [ ] Currency aligned
- [ ] Gross/net definitions aligned
- [ ] VAT treatment understood
- [ ] Business identifiers validated
- [ ] Blank identifiers quantified
- [ ] Duplicate behavior understood
- [ ] Exact matching performed
- [ ] Timing differences investigated
- [ ] Identifier variations investigated
- [ ] Amount differences quantified
- [ ] Record-level samples validated
- [ ] Reason codes evidence-based
- [ ] Residual cases visible
- [ ] Unresolved cases documented

---

## 44. Recommended Reconciliation Output

A professional reconciliation result should be able to answer:

```text
How many records exist in Source A?
How many exist in Source B?
How many match?
How many differ?
What is the financial difference?
Which differences are explained?
Which differences remain unresolved?
What evidence supports each explanation?
```

This is more useful than simply reporting:

```text
The numbers do not match.
```

---

## Final Principle

Reconciliation is not the process of forcing two systems to display the same number.

It is the process of moving from:

```text
Difference
```

to:

```text
Measured Difference
        ↓
Investigated Difference
        ↓
Classified Difference
        ↓
Explained Difference
        ↓
Documented Residual
```

The final result may still contain unresolved cases.

That does not automatically mean the reconciliation failed.

If the remaining uncertainty is quantified, documented, and communicated honestly, the analysis can still be reliable and useful.

> **The goal of reconciliation is not artificial equality. The goal is evidence-based understanding of why systems agree, why they differ, and what remains unresolved.**
