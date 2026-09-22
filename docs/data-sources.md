# Data Sources & Data Grain

This project combines multiple data sources that describe different stages of the Amazon order and financial process.

Understanding the grain, business meaning, identifiers, dates, and financial definitions of each source was essential before any reconciliation could be performed.

A central lesson from this project was:

> **Two systems can describe the same business activity while representing completely different analytical objects.**

For that reason, the sources were first analyzed independently before their values were compared.

---

## 1. Sellerboard

Sellerboard was used as an aggregated reporting source for Amazon performance.

The available monthly export contained metrics such as:

- Orders
- Organic Sales
- PPC Sales
- VAT
- Refunds
- Refund Costs
- Amazon Fees
- Shipping Costs
- Estimated Payout
- Cost of Goods
- Net Profit

### Data Grain

**Grain:** aggregated reporting period / month.

This distinction is important because the available Sellerboard export did not contain individual Order IDs.

Sellerboard could therefore be used for:

- monthly KPI comparison
- order-count comparison
- revenue comparison
- VAT analysis
- refund analysis
- high-level plausibility checks

However, it could not be used for direct order-by-order matching with the available monthly export.

### Important Financial Distinction

Sellerboard contained several financial metrics that should not be treated as interchangeable.

For example:

```text
Sales
Net-like Sales
Estimated Payout
Net Profit
```

represent different financial concepts.

A marketplace payout should therefore not automatically be compared with ERP sales revenue.

Before using a Sellerboard field in a reconciliation, its financial meaning must first be understood.

---

## 2. ERP / WaWi

The ERP/WaWi system contained detailed operational information about Amazon orders.

Relevant information included fields such as:

- External Order ID
- Purchase Date
- Order Creation Date
- Payment Date
- Shipping Date
- Order Status
- Cancellation Status
- Cancellation Timestamp
- Sales Channel
- Shipping Country
- Billing Country
- Currency
- Payment Information
- Order Positions
- Gross Values
- Net Values

### Data Grain

The ERP/WaWi environment contained information at both:

- order level
- order-line level

This distinction matters because one business order may contain several order positions.

Therefore:

```text
Order Count ≠ Order-Line Count
```

A row count from an order-line table should not automatically be interpreted as the number of customer orders.

### Role in the Reconciliation

The ERP/WaWi source was particularly important because individual orders could be investigated.

It allowed questions such as the following to be checked:

- Does this Order ID exist?
- Was the order cancelled?
- Was it shipped?
- When was it purchased?
- When was it created in the ERP?
- Which marketplace generated the order?
- Which country was the order shipped to?
- What was the gross value?
- What was the net value?
- Was payment information available?

This made the ERP/WaWi source essential for validating residual discrepancies after automated matching.

---

## 3. Marketplace vs. Shipping Country

One important finding during the analysis was that the shipping country could not safely be used as the Amazon marketplace.

For example:

```text
Sales Channel:     Amazon.de
Shipping Country:  France
```

does not mean that the order belongs to Amazon.fr.

Likewise:

```text
Sales Channel:     Amazon.fr
Shipping Country:  Germany
```

does not make the order an Amazon.de marketplace order.

The fields describe different concepts:

```text
Sales Channel     → Where the order originated
Shipping Country  → Where the order was delivered
```

For marketplace reconciliation, the dedicated sales-channel or marketplace field should therefore be used whenever available.

This was important when separating marketplaces such as:

- Amazon.de
- Amazon.fr
- Amazon.it
- Amazon.se
- Amazon.nl
- Amazon.pl

> **Destination does not prove marketplace origin.**

---

## 4. Amazon Settlement Data

Amazon settlement exports contained financial transactions related to marketplace activity.

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
- Tax on Shipping Credits
- Gift-Wrap Credits
- Promotional Discounts
- Tax on Promotional Discounts
- Marketplace Withheld Tax
- Selling Fees
- Fulfilment Fees
- Other Transaction Fees
- Other Adjustments
- Transaction Total

### Data Grain

**Grain:** financial transaction / settlement-line level.

This is fundamentally different from an ERP order table.

One customer order may generate multiple financial transaction rows.

For example, an order may later be associated with:

- product revenue
- VAT
- shipping credits
- fees
- refunds
- adjustments
- fulfilment charges

Therefore:

```text
Settlement Row Count ≠ Order Count
```

The business identifier must be analyzed before transaction rows are interpreted as orders.

---

## 5. Purchase Date vs. Settlement Date

One of the most important findings in the reconciliation was that the purchase period and settlement period should not automatically be expected to match.

An order may be purchased near the end of one month while the corresponding financial transaction appears in a later settlement period.

Therefore:

> **Purchase month ≠ Settlement month**

For example, conceptually:

```text
Purchase Date
     │
     ▼
Order Processing
     │
     ▼
Shipping / Payment Processing
     │
     ▼
Amazon Financial Transaction
     │
     ▼
Settlement Period
```

This means that searching only the original purchase month in settlement data can incorrectly classify valid orders as missing.

The reconciliation therefore searched appropriate later settlement periods before classifying an Order ID as a residual discrepancy.

---

## 6. Different Dates Represent Different Business Events

Several date fields existed across the sources.

Examples included:

```text
Purchase Date
Creation Date
Payment Date
Shipping Date
Refund Date
Settlement Transaction Date
```

These dates are not interchangeable.

They describe different stages of the business process.

A simplified interpretation is:

```text
Purchase Date             → Customer order event
ERP Creation Date         → Operational system event
Payment Date              → Payment event
Shipping Date             → Logistics event
Settlement Transaction    → Financial marketplace event
```

The correct date therefore depends on the analytical question.

Before comparing two systems, the reporting date definition must be explicit.

---

## 7. Order Status

Another important difference between systems was order status.

Operational Amazon data may contain both:

- shipped orders
- cancelled orders

If one system excludes cancelled orders while another source still contains records associated with those orders, the totals can differ without either source necessarily being incorrect.

Therefore, order reconciliation should explicitly define:

```text
What counts as a valid order?
```

Possible rules may include:

- include shipped orders
- exclude cancelled orders
- investigate partially processed orders separately

The correct rule depends on the business definition of the KPI.

---

## 8. Order IDs and Identifier Variations

Order IDs were one of the most important reconciliation keys.

However, identifier matching required additional validation.

Possible issues include:

- duplicate Order IDs
- blank Order IDs
- repeated transaction identifiers
- split transactions
- suffixed identifiers
- formatting differences

A conceptual example is:

```text
123-1234567-1234567
123-1234567-1234567_1
123-1234567-1234567_2
```

Such suffixes can help reveal relationships during exploratory analysis.

However, they should not automatically be removed in production.

The suffix may represent:

- split transactions
- partial processing
- adjustments
- partial shipments
- source-specific transaction logic

Therefore:

> **Analytical normalization is not automatically a valid production rule.**

The business meaning of an identifier variation should be validated before permanent transformation logic is introduced.

---

## 9. Gross, Net, VAT and Settlement Values

Financial fields also required careful interpretation.

Different systems may contain:

```text
Gross Sales
Net Sales
VAT
Refunds
Shipping Revenue
Discounts
Marketplace Fees
Settlement Amount
Estimated Payout
Profit
```

These values should not be compared simply because they are all monetary values.

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

The exact calculation depends on the source system.

The important rule is:

> **Revenue ≠ Settlement ≠ Payout ≠ Profit**

Before comparing financial KPIs, the financial definition of each source field must first be understood.

---

## 10. Source Comparison Matrix

A useful way to summarize the source differences is:

| Source | Typical Grain | Primary Use | Important Limitation |
|---|---|---|---|
| Sellerboard | Monthly aggregate | KPI and performance comparison | No individual Order IDs in the available monthly export |
| ERP/WaWi Orders | Order level | Operational order validation | Status and date definitions must be understood |
| ERP/WaWi Order Lines | Order-line level | Revenue and product analysis | Multiple rows can belong to one order |
| Amazon Settlement | Financial transaction level | Financial reconciliation | One order can generate multiple transactions and later-period activity |

This explains why the following comparison would be analytically unsafe:

```text
Sellerboard Orders = ERP Rows = Settlement Rows
```

The numbers describe different grains and potentially different business definitions.

---

## 11. Questions to Ask Before Comparing Sources

Before comparing two datasets, verify:

### Business Definition

- What exactly does the KPI represent?
- What counts as an order?
- Which records are excluded?

### Data Grain

- Is one row an order?
- Is one row an order position?
- Is one row a financial transaction?
- Is the source already aggregated?

### Identifier

- What is the business key?
- Is it unique?
- Can it appear multiple times?
- Are identifier variations possible?

### Date

- Which date controls the reporting period?
- Purchase date?
- Creation date?
- Shipping date?
- Payment date?
- Settlement date?

### Status

- Are cancelled orders included?
- Are refunds included?
- Are partially processed records included?

### Marketplace

- Which field identifies the marketplace?
- Is shipping country being confused with sales channel?

### Financial Definition

- Is the amount gross or net?
- Is VAT included?
- Are refunds included?
- Are shipping components included?
- Are marketplace fees included?
- Is the value revenue or payout?

### Currency

- Are all records in the same currency?
- Is currency conversion required?
- Which exchange rate would apply?

---

## 12. Why Data Grain Matters

A major lesson from this project was that data sources should not be compared only because they describe the same business process.

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
- transaction timing

Only after these definitions are aligned should remaining differences be interpreted as potential data-quality problems.

This prevents valid business differences from being incorrectly classified as:

- missing orders
- missing revenue
- broken imports
- incorrect Power BI calculations

---

## 13. Reconciliation Principle

The most important principle from the source analysis is:

> **A difference between two systems is evidence that requires investigation. It is not automatically evidence that one system is wrong.**

A professional reconciliation should determine whether the difference is caused by:

- business definition
- data grain
- reporting period
- status
- marketplace
- identifier structure
- financial definition
- settlement timing
- transformation logic
- Power BI filter context
- or a genuinely unresolved source discrepancy

Only after those possibilities have been investigated should a final conclusion be made.

---

## Final Takeaway

Understanding the source systems was more important than immediately writing complex DAX.

The reconciliation became much easier once each dataset could answer four basic questions:

```text
What does one row represent?
What business event does the date represent?
What does the identifier represent?
What does the financial value represent?
```

Only after those questions were answered could the sources be compared meaningfully.

> **Two datasets describing the same business process do not necessarily represent the same analytical object.**
