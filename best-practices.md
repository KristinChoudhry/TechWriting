# Product Catalog: Best Practices

This article provides best practices for configuring and maintaining your Product Catalog to ensure accurate billing and scalable operations. Following these practices improves data integrity, simplifies reporting, and reduces errors.

## Standardize naming conventions

Establish consistent names for SKUs, descriptions, and attributes to improve searchability and reduce ambiguity. Define a pattern such as `CATEGORY-FEATURE-TIER` and document it.

Ignoring consistency can cause duplicate items, billing mistakes, and reporting confusion.

## Regularly audit product configurations

Schedule audits to remove deprecated items, align pricing across price books, and verify tax rules and statuses. Audits prevent stale data and compliance issues.

## Separate pricing logic from product definitions

Use price books and pricing rules instead of embedding logic directly in product attributes. This increases flexibility and simplifies updates.

### Common pitfalls to avoid

- Pitfall: Overly complex per‑product pricing
  - Why it’s a problem: Difficult to maintain; error‑prone
  - Recommendation: Centralize logic in pricing rules/engines

- Pitfall: Uncontrolled catalog growth
  - Why it’s a problem: Confuses users; slows operations
  - Recommendation: Introduce governance, archival, and clear naming

Key takeaways: standardize names, audit regularly, and centralize pricing logic.
