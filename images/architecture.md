# Data Architecture

This diagram shows the high-level reconciliation architecture used in this Power BI case study.

```mermaid
flowchart LR
    A[Amazon Marketplace]
    B[Sellerboard]
    C[Amazon Settlement Data]
    D[ERP / WaWi]

    A --> E[Power Query]
    B --> E
    C --> E
    D --> E

    E --> F[Data Cleaning & Normalization]
    F --> G[Validation & Reconciliation Layer]

    G --> H[Order Reconciliation]
    G --> I[Revenue Reconciliation]
    G --> J[Settlement Validation]

    H --> K[Power BI Semantic Model]
    I --> K
    J --> K

    K --> L[DAX Measures]
    L --> M[Power BI Reporting]
```

## Reconciliation Principle

The architecture does not assume that different source systems must produce identical values.

Instead, the reconciliation layer is used to identify, quantify, and explain differences caused by factors such as:

- different order definitions
- cancellations
- split transactions
- settlement timing
- gross vs. net revenue
- VAT treatment
- inconsistent date contexts
- source-specific business rules

The objective is a transparent and traceable reconciliation process rather than artificially forcing the source systems to match.
