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

> A numerical difference between two systems does not automatically mean that data is missing or incorrect.

Before correcting any KPI, the underlying business definition, data grain, date logic, order status, marketplace, and revenue definition must first be understood.

### Core questions

The reconciliation therefore focused on questions such as:

1. What exactly counts as an order in each system?
2. Which date determines the reporting period?
3. How should cancelled orders be handled?
4. Are order IDs truly comparable across the systems?
5. Which marketplace does an order belong to?
6. Are the compared revenue values gross, net, settlement, or payout values?
7. How are VAT, refunds, fees, discounts, and shipping treated?
8. Can settlement transactions occur in a later month than the original purchase?
9. Is a Power BI difference caused by the source data, the data model, or the DAX filter context?
10. When is a remaining difference a genuine data-quality issue rather than a definitional difference?

These questions became the foundation of the reconciliation methodology used throughout this project.
