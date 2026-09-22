# Lessons Learned

This project started as a Power BI reporting task.

It became a much broader lesson in data analysis, reconciliation, data quality, business definitions, source-system behavior, and analytical discipline.

The most important realization was that building a dashboard is often not primarily a visualization problem.

Before a KPI can be trusted, an analyst may need to understand:

- where the data comes from
- what one row represents
- which business event a date represents
- which records belong to the KPI
- how identifiers behave
- how financial values are defined
- how Power BI propagates filters
- why different systems legitimately report different values
- which differences can be explained
- which differences remain unresolved

The following lessons summarize the most important insights from the project.

---

## 1. Understand the Business Definition Before Writing DAX

A technically correct measure can still answer the wrong business question.

For example:

```text
Orders
Revenue
Open Orders
Net Sales
```

sound like clear KPIs.

In reality, each requires a definition.

For **Orders**, questions include:

- Are cancelled orders included?
- Which date determines the reporting period?
- Which marketplace is included?
- Is one row one order?
- Are duplicate Order IDs possible?

For **Revenue**, questions include:

- Gross or net?
- VAT included or excluded?
- Refunds included?
- Shipping included?
- Fees included?
- Which order population is used?

The first step should therefore be:

> **Define the KPI before calculating the KPI.**

---

## 2. Data Grain Is One of the First Things to Check

Many reconciliation problems begin because tables are compared at different levels of detail.

Examples:

```text
Monthly KPI
Order
Order Line
Settlement Transaction
Refund Transaction
```

These are different analytical objects.

Therefore:

```text
Row Count ≠ Order Count
```

and:

```text
Settlement Transaction ≠ Customer Order
```

Before counting, joining, or creating relationships, determine:

> **What does one row represent?**

This simple question prevents many downstream errors.

---

## 3. A Difference Is Not Automatically an Error

One of the most important lessons from the project was to avoid interpreting every numerical difference as missing or incorrect data.

If:

```text
System A = 1,000 Orders
System B =   970 Orders
```

the immediate conclusion should not be:

```text
30 orders are missing.
```

The only confirmed statement is:

```text
The sources differ by 30 under the current definitions.
```

Possible explanations include:

- different status rules
- different periods
- marketplace scope
- settlement timing
- identifier differences
- refresh timing
- different business definitions

A neutral term such as:

> **Source difference**

is more appropriate until the cause is demonstrated.

---

## 4. Source Systems Do Not Need to Agree Automatically

Different systems exist for different purposes.

For example:

```text
ERP / WaWi
    → operational processing

Marketplace Reporting
    → performance reporting

Settlement Data
    → financial transaction processing
```

Even when all systems describe the same commercial activity, they may not represent it in the same way.

Therefore, exact equality should never be assumed before the definitions are aligned.

The analyst's job is not to force equality.

The analyst's job is to explain the difference.

---

## 5. Purchase Date and Settlement Date Are Different Events

A customer purchase and a financial settlement transaction belong to different stages of the process.

Conceptually:

```text
Purchase
   ↓
Order Processing
   ↓
Payment
   ↓
Shipping
   ↓
Financial Transaction
   ↓
Settlement
```

These events can occur in different reporting periods.

Therefore:

> **Purchase month ≠ Settlement month**

Searching only the purchase month in settlement data can create false unmatched orders.

Timing should be investigated before records are classified as missing.

---

## 6. Marketplace and Shipping Country Are Different Concepts

One important source-level lesson was that destination does not determine marketplace origin.

For example:

```text
Marketplace:      Amazon.de
Shipping Country: France
```

can be valid.

Therefore:

```text
Shipping Country ≠ Marketplace
```

The dedicated sales-channel or marketplace field should be used for marketplace analysis whenever available.

---

## 7. Identifiers Need Business Understanding

An Order ID looks like a technical field, but its behavior can contain business meaning.

Identifier variations may represent:

- split transactions
- partial shipments
- adjustments
- partial processing
- source-specific logic

For example:

```text
ORDER-123
ORDER-123_1
ORDER-123_2
```

Removing suffixes may help exploratory matching.

However:

> **Analytical normalization is not automatically valid production logic.**

Before changing identifiers permanently, understand why the variation exists.

---

## 8. Exact Matching Should Come Before Assumptions

Structured business identifiers should normally be matched exactly first.

A useful sequence is:

```text
Exact Match
    ↓
Investigate Unmatched Records
    ↓
Check Timing
    ↓
Check Identifier Variations
    ↓
Controlled Normalization
    ↓
Residual Cases
```

This creates a traceable process and reduces the risk of creating false matches.

---

## 9. Revenue Is Not One Universal Number

Financial reconciliation taught an important lesson:

```text
Revenue
Settlement
Payout
Profit
```

are different concepts.

Likewise:

```text
Gross Sales
Net Sales
VAT
Refunds
Fees
Shipping
```

must be understood before they are combined.

Therefore:

> **Revenue ≠ Settlement ≠ Payout ≠ Profit**

A field should not be selected for comparison simply because it contains a monetary value.

---

## 10. Gross and Net Must Be Explicit

Before comparing revenue, determine whether each source reports:

```text
Gross
Net
VAT
```

and how those values interact.

A plausible equation such as:

```text
Gross = Net + VAT
```

can be useful for validation.

But marketplace data may also contain:

- different VAT rates
- shipping VAT
- discounts
- refunds
- tax adjustments
- marketplace tax rules
- rounding differences

Therefore, financial formulas must be validated against actual source behavior.

---

## 11. Payout Should Not Be Used as a Sales Comparator

Marketplace payout values may include:

- revenue
- refunds
- fees
- fulfilment costs
- reimbursements
- adjustments
- previous-period activity

Therefore:

```text
ERP Revenue
```

should not automatically be compared with:

```text
Marketplace Payout
```

A large difference may simply mean that two different financial concepts were compared.

---

## 12. Use the Same Population for Related KPIs

If an order KPI excludes cancelled orders, its related revenue KPI should normally use the same intended order population.

Otherwise:

```text
Orders  → Population A
Revenue → Population B
```

while the dashboard makes them appear related.

A better design is:

```text
Validated Order Population
          │
          ├──► Order KPI
          │
          └──► Revenue KPI
```

Consistency of population is essential for meaningful KPI interpretation.

---

## 13. Power BI Problems Are Often Context Problems

A measure can be mathematically correct and still return the wrong result.

The cause may be:

- wrong slicer source
- wrong date table
- missing relationship
- inactive relationship
- unexpected filter direction
- `ALL` removing context
- disconnected tables
- page-specific filters

Therefore:

> **Wrong result does not automatically mean wrong arithmetic.**

Before rewriting DAX, inspect the filter context.

---

## 14. The Same Measure Can Behave Differently on Different Pages

A management page may use:

```text
Calendar[Date]
```

while a source-analysis page uses:

```text
MonthlyReport[Period]
```

A measure written for the first context may not receive the correct period on the second page.

This was an important practical lesson:

> **Reusable DAX still needs reusable context.**

A measure should be validated wherever it is used.

---

## 15. Build Small Control Measures

Complex measures are easier to debug when the problem is divided into smaller questions.

Instead of immediately debugging:

```text
Final Management Revenue
```

first validate:

```text
Raw Orders
Valid Orders
Canceled Orders
Selected Period
Marketplace Population
Base Revenue
Filtered Revenue
```

Each control measure should answer one question.

This makes it easier to identify the first point where the result becomes unexpected.

---

## 16. Change One Thing at a Time

Changing several elements simultaneously creates uncertainty.

For example:

```text
Change Relationship
Change DAX
Change Date Logic
Change Status Filter
```

and then seeing the correct result does not reveal which change actually solved the problem.

A stronger analytical process is:

```text
Question
   ↓
Hypothesis
   ↓
One Test
   ↓
Evidence
   ↓
Conclusion
   ↓
Next Question
```

This is slower for one minute and much faster over an entire project.

---

## 17. Reduce the Problem Before Manual Investigation

Manual validation is useful, but it should not be the first step.

Instead of manually checking thousands of orders:

```text
All Records
    ↓
Correct Period
    ↓
Correct Status
    ↓
Correct Marketplace
    ↓
Exact ID Matching
    ↓
Settlement Timing
    ↓
Identifier Analysis
    ↓
Residual Records
```

Only the residual population should normally require detailed manual investigation.

This makes the process more scalable and reproducible.

---

## 18. One Test Case Proves One Test Case

Finding one order that matches perfectly is useful.

It confirms that the logic works for that record.

It does not prove:

```text
The entire dataset is correct.
```

Likewise, one incorrect record does not prove that the entire source is wrong.

Conclusions should match the scope of the evidence.

> **This test case confirms this case.**

Broader claims require broader validation.

---

## 19. Aggregate Totals Can Hide Record-Level Problems

Two aggregate totals can match while individual records are wrong.

For example:

```text
Order A Difference   +50
Order B Difference   -50
-------------------------
Total Difference       0
```

The total looks perfect.

The underlying data is not.

Therefore, important reconciliations should validate:

- aggregate totals
- record-level matches

Both views provide different evidence.

---

## 20. A Plausible Number Is Not Validation

A KPI may look reasonable because it is close to expectations.

That does not prove correctness.

Validation should use evidence such as:

- source totals
- distinct counts
- status distributions
- marketplace distributions
- individual records
- independent control measures

> **Plausibility is useful for detecting problems, but it is not proof.**

---

## 21. Do Not Hide Differences With Correction Factors

A correction factor can make two totals look identical without explaining the discrepancy.

It can hide:

- wrong populations
- wrong filters
- marketplace differences
- VAT problems
- missing transactions
- source-system behavior

Correction logic should only be used when there is a documented and defensible business rule.

It should not be used simply to make a dashboard look correct.

---

## 22. Unresolved Is a Valid Result

Not every discrepancy can always be explained with the available data.

A professional analyst should be able to say:

```text
Unresolved
```

when the evidence does not support a stronger conclusion.

This is better than inventing certainty.

A useful classification is:

```text
Confirmed
Empirically Supported
Hypothesis
Unresolved
```

---

## 23. Separate Facts From Hypotheses

During investigation, it is easy for a plausible explanation to become treated as fact.

For example:

```text
"The order is probably missing because of settlement timing."
```

is a hypothesis.

After finding the order in a later settlement period:

```text
"The order appears in a later settlement period."
```

becomes a confirmed observation.

This distinction improves analytical quality and documentation.

---

## 24. Keep an Investigation Log

Long reconciliation projects can easily repeat the same investigation.

A simple log can contain:

```text
Issue
Expected Result
Actual Result
Hypothesis
Test
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

This creates an audit trail and prevents repeated troubleshooting loops.

---

## 25. Preserve Raw Data

Aggressive cleaning can make data easier to use while making problems harder to understand.

A better pattern is:

```text
Raw
 ↓
Staging
 ↓
Clean
 ↓
Model
```

The original source should remain traceable whenever practical.

This allows an analyst to determine whether a problem originated:

- in the source
- in transformation
- in the model
- in DAX

---

## 26. Power Query Should Be Traceable

Transformation steps should have a clear reason.

Operations such as:

- removing rows
- removing duplicates
- replacing values
- changing types
- filtering records

can alter reconciliation results.

Important transformations should therefore be understandable and documented.

A useful rule is:

> **Never silently remove a record you may later need to explain.**

---

## 27. Data Quality Is Part of Analysis

Data quality is not separate from business intelligence.

Problems such as:

- missing IDs
- duplicate records
- unexpected statuses
- invalid dates
- inconsistent marketplace values
- mixed currencies

directly affect KPI interpretation.

Therefore, data-quality checks should be part of the analytical workflow rather than an afterthought.

---

## 28. Source Freshness Matters

Two systems may temporarily disagree simply because they were refreshed at different times.

Before escalating a discrepancy, check:

```text
When was Source A refreshed?
When was Source B refreshed?
Is settlement processing delayed?
Did an import fail?
```

A source difference can be a synchronization issue rather than a business-data issue.

---

## 29. API Completeness Must Be Verified

A successful API request does not automatically mean that the complete dataset was downloaded.

Potential risks include:

- pagination
- result limits
- rate limits
- failed requests
- incomplete retries

A technically successful pipeline can therefore still contain incomplete data.

Row counts and completeness checks should be monitored.

---

## 30. Historical Data Can Change

Orders are not always static after their initial creation.

Historical records can later receive:

- cancellation
- refund
- payment changes
- shipping updates
- financial adjustments

Incremental refresh strategies must therefore account for the possibility that older records change.

Otherwise, the Power BI model can preserve outdated history.

---

## 31. Schema Drift Is a Real Production Risk

Source systems evolve.

A future export may contain:

- renamed columns
- new columns
- new status values
- new transaction types
- changed data types

A report can continue refreshing while silently producing incomplete business logic.

Production solutions should therefore monitor source structure as well as refresh success.

---

## 32. Documentation Is Part of the Solution

A dashboard is difficult to maintain when important logic exists only in the original analyst's memory.

Important documentation should explain:

- source
- grain
- business key
- date definition
- KPI population
- marketplace scope
- financial definition
- exclusions
- known limitations

Good documentation allows another analyst to understand not only **what** was calculated, but **why**.

---

## 33. Management Reporting and Reconciliation Serve Different Purposes

Management usually needs simple KPIs.

For example:

```text
Orders
Revenue
Open Orders
Shipping
```

A reconciliation page needs deeper diagnostics:

```text
Source A
Source B
Difference
Matched
Cancelled
Timing Difference
Unresolved
```

Trying to place all technical diagnostics on the management page reduces readability.

A better design separates:

```text
Operational Reporting
```

from:

```text
Analytical Reconciliation
```

---

## 34. Transparency Is Better Than Artificial Precision

A dashboard does not become more professional because every source shows exactly the same number.

It becomes more professional when users can understand:

- where the KPI came from
- how it was defined
- why another source differs
- how large the difference is
- whether the cause is known

Transparent differences are more valuable than unexplained equality.

---

## 35. Technical Skill Alone Is Not Enough

Power BI reconciliation requires more than knowing DAX syntax.

The analyst must combine:

```text
Business Understanding
        +
Data Modeling
        +
Data Quality
        +
Power Query
        +
DAX
        +
Validation
        +
Communication
```

A technically sophisticated formula cannot compensate for a misunderstood business definition.

---

## 36. Ask Better Questions

Early in a reconciliation project, the natural question is often:

> **Why do these numbers not match?**

A better sequence is:

```text
What does Source A represent?
What does Source B represent?
What is the grain?
What is the business key?
Which period is used?
Which statuses are included?
Which marketplace is included?
Which financial definition is used?
Only then: why is there a difference?
```

Better questions reduce unnecessary debugging.

---

## 37. Evidence Should Determine the Conclusion

A strong analyst should be willing to change the hypothesis when the data contradicts it.

The process should be:

```text
Hypothesis
    ↓
Test
    ↓
Evidence
    ↓
Conclusion
```

not:

```text
Desired Conclusion
       ↓
Search for Supporting Evidence
```

Reconciliation should be evidence-driven.

---

## 38. Do Not Confuse Technical Success With Business Correctness

Power BI may allow:

- a relationship
- a calculated column
- a measure
- a many-to-many model
- a filter

without any technical error.

That does not mean the model represents the business correctly.

> **Technically valid does not automatically mean analytically valid.**

Business validation remains necessary.

---

## 39. Know When to Stop Investigating

Reconciliation can become endless if every tiny residual difference must be explained at any cost.

A reasonable stopping point may be reached when:

- major discrepancy drivers are understood
- remaining differences are quantified
- available data has been exhausted
- unresolved cases are documented
- further investigation requires another system or business owner

At that point, the correct result may be:

```text
Known Difference
Cause: Unresolved
Further Validation Required
```

This is more professional than inventing an explanation.

---

## 40. What I Would Do Earlier in the Next Project

If starting a similar project again, the following steps should happen earlier.

### Create a Source Dictionary

Document:

```text
Source
Table
Field
Business Meaning
Data Type
Grain
Known Limitations
```

### Create a KPI Dictionary

Document:

```text
KPI
Definition
Source
Date
Population
Exclusions
Financial Treatment
```

### Create a Reconciliation Table Early

Instead of investigating differences only through separate visuals, create a controlled record-level comparison layer.

### Create Control Measures Before Final KPIs

Validate the population before building management measures.

### Separate Reporting and Diagnostics

Keep the management dashboard simple while maintaining a dedicated reconciliation view.

### Maintain an Issue Log

Record hypotheses, tests, evidence, and unresolved cases.

These steps would reduce repeated debugging later.

---

## 41. What Beginners Should Know Before Their First Reconciliation Project

A beginner may expect the main challenge to be writing DAX.

Often it is not.

The harder questions are:

```text
What does this field actually mean?
Why does this table have several rows per order?
Which date should I use?
Why does another system count differently?
Is this amount gross or net?
Why does the measure change on another page?
Is the difference actually an error?
```

Learning to answer these questions is a major part of becoming a stronger data analyst.

---

## 42. A Practical Analyst Mindset

A useful working mindset is:

```text
Do not guess.
      ↓
Inspect the data.
      ↓
Define the business meaning.
      ↓
Build one test.
      ↓
Evaluate the evidence.
      ↓
Document the result.
      ↓
Continue only when necessary.
```

This reduces trial-and-error work and produces more defensible conclusions.

---

## 43. Reusable End-to-End Method

The complete methodology from this project can be summarized as:

```text
1. Understand the business question
              ↓
2. Inventory the source systems
              ↓
3. Determine data grain
              ↓
4. Identify business keys
              ↓
5. Define relevant dates
              ↓
6. Define KPI populations
              ↓
7. Validate data quality
              ↓
8. Build simple control measures
              ↓
9. Align marketplace and currency
              ↓
10. Reconcile Order IDs
              ↓
11. Reconcile financial values
              ↓
12. Investigate timing differences
              ↓
13. Investigate residual records
              ↓
14. Validate Power BI filter context
              ↓
15. Classify evidence
              ↓
16. Document unresolved differences
              ↓
17. Build transparent reporting
              ↓
18. Monitor the solution over time
```

---

## 44. The Most Important Lessons

If only a few principles are remembered from this project, they should be these:

1. **Define the KPI before calculating it.**
2. **Understand the grain before counting rows.**
3. **A difference does not automatically mean missing data.**
4. **Purchase date and settlement date describe different events.**
5. **Shipping country is not marketplace.**
6. **Revenue, settlement, payout, and profit are different concepts.**
7. **Use the same business population for related KPIs.**
8. **Check filter context before rewriting DAX.**
9. **One test case validates one test case.**
10. **Do not hide unexplained differences with correction factors.**
11. **Keep unresolved cases visible.**
12. **Let evidence determine the conclusion.**

---

## Final Reflection

The most valuable result of this project was not one specific DAX measure.

It was the analytical process required to move from:

```text
"The numbers do not match."
```

to:

```text
"We understand what each number represents,
which differences are explained,
which assumptions have been validated,
and which residual differences remain unresolved."
```

That distinction is fundamental to professional data analysis.

A dashboard is trustworthy not because every number looks clean, but because its definitions, transformations, limitations, and remaining uncertainty can be explained.

> **Good data analysis does not force the data to tell the expected story. It builds the strongest conclusion that the available evidence can support.**
