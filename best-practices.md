# Product Catalog: Best Practices

This article provides best practices for configuring and maintaining your Product Catalog to ensure accurate billing and scalable operations. Following these practices improves data integrity, simplifies reporting, and reduces errors.

---

## Standardize naming conventions

Establish consistent names for SKUs, descriptions, and attributes to improve searchability and reduce ambiguity. Define a pattern such as `CATEGORY-FEATURE-TIER` and document it.

- **Define a canonical pattern**
  - Example patterns:
    - `CATEGORY-FEATURE-TIER` (for example, `STORAGE-SSD-PREMIUM`)
    - `FAMILY:SUBFAMILY:VARIANT` (for example, `VOICE:INTL:PAYG`)
  - Reserve specialized suffixes for lifecycle and packaging: `-LEGACY`, `-BUNDLE`, `-ADDON`.
- **Codify identifiers**
  - SKU/code: stable, short, unique, machine-friendly (no spaces; use `A-Z`, `0-9`, `-`).
  - Display name: human-friendly, localized as needed.
  - Description: first line concise; second line details usage, units, and constraints.
- **Attribute naming**
  - Use consistent prefixes for rating inputs: `rate_`, `limit_`, `threshold_`.
  - Use units in attribute names or document units explicitly (for example, `rate_per_gb`).
- **Documentation and governance**
  - Maintain a naming guideline page with examples, reserved tokens, and anti-patterns.
  - Add a pre-commit or CI check that flags non-conforming names.
- **Localization**
  - Keep SKU immutable; localize display names and descriptions via translation keys.

Ignoring consistency can cause duplicate items, billing mistakes, and reporting confusion.

---

## Regularly audit product configurations

Schedule audits to remove deprecated items, align pricing across price books, and verify tax rules and statuses. Audits prevent stale data and compliance issues.

- **Audit cadence**
  - Monthly: price alignment, tax flags, status checks.
  - Quarterly: full catalog review, bundling rules, attribute drift, and metadata completeness.
  - Pre-release: targeted audit for upcoming launch changes.
- **Audit checklist**
  - Status hygiene: only sellable items are Active; sunset Inactive/Deprecated correctly.
  - Price parity: cross–price book variance within policy thresholds.
  - Effective dating: no overlaps or gaps for the same product/region/currency.
  - Taxability: correct tax category and nexus coverage; verify VAT/GST attributes.
  - Accounting mappings: GL accounts, revenue recognition rules, and SKU-to-ledger mapping present.
  - Attribute validation: required rating inputs present; units documented.
  - Bundles and dependencies: included items exist and quantities match packaging rules.
  - Permissions and visibility: channel, region, and segment scoping match go-to-market plan.
- **Tools and telemetry**
  - Use reports to list orphaned products, duplicate names, and unreferenced attributes.
  - Implement validation rules that block activation if required data is missing.
- **Remediation workflow**
  - Triage issues by severity, assign owners, track SLAs, and require peer review for fixes.

---

## Separate pricing logic from product definitions

Use price books and pricing rules instead of embedding logic directly in product attributes. This increases flexibility and simplifies updates.

- **Do**
  - Keep products as descriptive containers: identity, taxonomy, and capability.
  - Store rates, discounts, and eligibility in price books and rules engines.
  - Use effective-dated rate tables for time-based changes.
- **Don’t**
  - Hardcode discounts or formulas in product descriptions or ad hoc attributes.
  - Clone products to represent temporary promotions.
- **Benefits**
  - Faster updates without product churn.
  - Cleaner audit trails and simpler rollback.
  - Reduced risk of inconsistent pricing across channels.
- **Implementation tips**
  - Establish a canonical price book hierarchy: Global Base → Regional → Segment → Contract.
  - Define precedence rules and conflict resolution (for example, lowest price wins, or specific book overrides base).
  - Version rules and attach change reasons for compliance.

---

## Common pitfalls to avoid

### Overly complex per-product pricing

- **Why it’s a problem**: Difficult to maintain; error-prone; inhibits experimentation; complicates audits.
- **Recommendation**: Centralize logic in pricing rules/engines. Use a small set of reusable rule templates, and feed them product attributes intentionally (for example, `rate_per_unit`, `min_commitment`). Version rules and test with representative scenarios before rollout.

### Uncontrolled catalog growth

- **Why it’s a problem**: Confuses users; slows operations; increases reporting complexity; fragments analytics.
- **Recommendation**: Introduce governance, archival, and clear naming. Use a product request intake form, require justification for new SKUs, prefer parameterization over cloning, and set lifecycle states with target deprecation dates. Run quarterly SKU rationalization to merge or retire low-usage items.

### Missing effective dating and overlaps

- **Why it’s a problem**: Causes mid-period price flips, double-charging, or unintended gaps.
- **Recommendation**: Enforce non-overlapping ranges in validation; require `effective_start` and optional `effective_end`; provide a preview tool to evaluate pricing at boundary times.

### Inconsistent tax and accounting mappings

- **Why it’s a problem**: Leads to tax miscalculation, failed postings, and month-end reconciliation issues.
- **Recommendation**: Tie each SKU to a tax category and GL account at activation. Automate checks against a master mapping table. Require finance review for tax-sensitive changes.

### Region/segment visibility misconfiguration

- **Why it’s a problem**: Customers see the wrong products or prices; channel conflicts.
- **Recommendation**: Use explicit scopes on products and price books. Test visibility with impersonation for key segments and regions.

---

## Key takeaways: standardize names, audit regularly, and centralize pricing logic.

- **Standardize**: Enforce naming and attribute patterns to eliminate ambiguity and improve search and analytics.
- **Audit**: Run scheduled audits with validation rules to prevent stale data, overlaps, and compliance gaps.
- **Centralize**: Keep pricing logic in rules and price books, not in product records, to scale safely and update quickly.

Additional recommendations:
- **Lifecycle management**: Use Draft → Active → Inactive → Deprecated with clear entry/exit criteria and communication to downstream teams.
- **Change control**: Require peer review for catalog and pricing changes; deploy via change sets with rollback plans.
- **Observability**: Track catalog KPIs (active SKU count, orphaned rates, overlap errors, failed validations) and alert on regressions.
- **Documentation**: Maintain a living catalog playbook with naming rules, audit checklists, pricing hierarchy, and approval workflows.
