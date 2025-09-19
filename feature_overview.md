# Product Catalog

The Product Catalog centralizes all sellable items, bundles, and pricing logic. It helps teams launch, manage, and update offers consistently across markets.

This feature reduces manual configuration, enforces consistency, and accelerates time-to-market. It is most useful for billing administrators and product managers.

## Streamlining your product offerings

Creating products from scratch for each variation is time-consuming and error-prone. The Product Catalog solves this by:
- Standardizing product structures, attributes, and pricing rules.
- Supporting reusable components and bundles.
- Enforcing governance via roles and approvals.

Additional capabilities:
- Parameterized products with attributes such as `rate_per_unit`, `term_length`, and `feature_flags` to avoid SKU proliferation.
- Effective-dated pricing and status controls to manage launches, promotions, and sunsets cleanly.
- Validation rules to prevent missing tax categories, GL codes, or required rating inputs before activation.
- Visibility scoping by region, channel, and segment so users only see relevant items.

If you work with entities, note:
- Account represents your customer. Accounts can exist in hierarchies and are used for subscriptions and billing.
- Product is the sellable unit. Products may include options, add-ons, and dependencies.
- Price Book stores rates and discounts associated to products for regions, segments, or contracts.
- Subscription references the chosen product, term, and pricing at the time of sale.

Operational guidance:
- Use Draft → Active → Inactive → Deprecated states with clear criteria.
- Prefer bundles for packaged offers; use quantity and eligibility rules to keep configurations consistent.
- Track changes with reason codes and audit logs for compliance and rollback.

## Related features and integrations

- Subscription Plan Cloning — quickly derive new offers from existing plans.
- Revenue Recognition — link products to revrec policies.
- Price Books — manage region- or customer-specific price variations.
- Tax Engine — assign tax categories and nexus rules to products for accurate calculation.
- Promotions — schedule time-bound discounts that layer on top of standard pricing.
- Reporting and Analytics — expose catalog metrics such as active SKU count, adoption by segment, and pricing variance.

## Summary

The Product Catalog provides a single source of truth for offers and pricing, improving speed and accuracy. For deeper guidance, see:
- Configure Subscription Plans
- API: Products
- Best Practices: Product Modeling
- Price Books and Rules
- Tax Configuration and Compliance Notes

