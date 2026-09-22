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



