# Rate Loader: Common Use Cases

The Rate Loader supports bulk rate management scenarios for pricing and billing teams. This article covers updating rates, applying promotions, and managing tiered pricing. These use cases improve accuracy and speed while reducing manual effort. Intended readers include Billing Administrators and Pricing Managers.

---

## Apply updated rates

Use this when you need to change base prices across many products at once (for example, an annual price list update or currency realignment). Supports effective dating so future changes can be staged ahead of time.

Prerequisites:
- You have edit permissions for rates.
- Target products already exist with valid `product_code` entries.
- If you use multi-currency, confirm that `currency` codes match your ISO currency setup.

Data preparation:
- Required CSV headers: `product_code`, `new_rate`, `currency`, `effective_start`
- Optional headers:
  - `effective_end` to end-date the current rate
  - `reason_code` for audit trails (for example, `ANNUAL_REPRICE`, `COST_PASS_THROUGH`)
  - `rounding_rule` if your system supports per-item rounding overrides (for example, `NEAREST_0_01`)
  - `notes` for internal change history

Field rules:
- `product_code`: Exact match to an existing product. Case-sensitive unless configured otherwise.
- `new_rate`: Decimal up to 4 places. Must be ≥ 0.
- `currency`: ISO 4217 code (for example, `USD`, `EUR`). Must be enabled in your tenant.
- `effective_start`: Date or datetime in your accepted format (for example, `YYYY-MM-DD` or `YYYY-MM-DDTHH:MM:SSZ`).
- `effective_end` (optional): If supplied, must be > `effective_start` and non-overlapping with existing active rates.

Steps:
1. Prepare CSV: `product_code, new_rate, currency, effective_start[, effective_end, reason_code, rounding_rule, notes]`
2. Open Standard Rate Loader and create a new upload.
3. Upload the CSV and map fields.
   - Map dates to the correct format and timezone.
   - Verify numeric columns detect as decimal/currency.
4. Validate changes.
   - Loader flags missing products, invalid currency codes, overlapping dates, and negative rates.
   - Resolve flagged rows and re-upload, or use the inline correction panel if available.
5. Click Apply Updates to confirm.
   - Choose conflict handling:
     - Overwrite existing effective-dated row
     - Insert a new row and end-date the current one
     - Skip on conflict
   - Optionally schedule execution at a future time to avoid peak hours.

Post-apply checks:
- Spot-check a sample using rate preview for today and the next change window.
- Export an audit report filtered by `reason_code` to archive the change set.

---

## Apply promotional rates

Run time-bound discounts for targeted products. Use this for seasonal campaigns, market tests, or account-specific offers that still live in the standard rate catalog.

Scope suggestions:
- Limit to specific `product_code` sets or categories.
- Define guardrails such as minimum price floors or required approval.

Scheduling considerations:
- Use precise start and end datetimes to avoid boundary errors.
- Align timezone with storefront/quote engine timezone to prevent early or late activation.

Schedule a promotion

Data preparation:
- Required CSV headers: `product_code`, `promo_rate`, `promo_start`, `promo_end`
- Optional headers:
  - `promo_code` for campaign tracking
  - `max_quantity` to cap promo usage per transaction
  - `segment` to target customer groups if your catalog supports segmentation
  - `stacking_rule` to define interaction with other discounts (for example, `NO_STACK`, `STACK_PERCENT`, `STACK_LOWEST_PRICE`)
  - `reason_code` (for example, `PROMO_SPRING_2025`)
  - `notes`

Field rules:
- `promo_rate`: Must be ≤ the standard rate in effect during the promo window unless override policy allows exceptions.
- `promo_start` / `promo_end`: Non-overlapping with other promos for the same product unless stacking is explicitly allowed.
- If both standard and promo rates are present, the pricing engine should select the lower eligible rate, subject to `stacking_rule`.

Steps:
1. Prepare CSV: `product_code, promo_rate, promo_start, promo_end[, promo_code, max_quantity, segment, stacking_rule, reason_code, notes]`
2. Upload and map fields, including dates.
   - Confirm datetime format and timezone mapping.
   - Map `stacking_rule` to a recognized enumeration.
3. Validate and set the schedule to the campaign start time.
   - Resolve conflicts such as overlapping promos, price floors, and unknown segments.
4. Confirm the promotion and schedule activation.
   - Optionally enable automatic reversion to standard rates at `promo_end`.
   - Notify downstream teams (sales, support) with the audit export.

Monitoring:
- Use the rate preview tool to verify effective pricing at boundary times \(promo_start - 1 minute, promo_start, promo_end, promo_end + 1 minute\).
- Track uplift metrics using `promo_code` in analytics.

---

## Manage tiered pricing

Adjust tiers for usage-based or volume-based models. Use this for graduated or volume discounts where unit price depends on quantity or consumption.

Tier model types:
- Graduated (tiered): Each tier has a quantity band with its own unit rate; charges sum across tiers consumed.
- Volume: A single unit rate applies to all units based on the band the total falls into.

Data integrity rules:
- Tiers must be contiguous and non-overlapping.
- First tier `tier_min` must be 0 or 1 depending on your convention.
- The last tier `tier_max` can be `NULL`/blank to represent open-ended.
- Rates must be non-negative and adhere to rounding policy.

Update pricing tiers

Data preparation:
- Required CSV headers: `product_code`, `tier_min`, `tier_max`, `rate`
- Optional headers:
  - `model_type` with allowed values `GRADUATED` or `VOLUME` (defaults to your system’s standard if omitted)
  - `currency` if multi-currency tiers are supported
  - `effective_start` and `effective_end` for time-bounded tier tables
  - `reason_code`, `notes`

Field rules:
- `tier_min` and `tier_max`: Integers where `tier_min ≤ tier_max` unless `tier_max` is open-ended.
- Tiers must start at the lower boundary and have no gaps: for example, 0–99, 100–499, 500–∞.
- For `model_type = VOLUME`, ensure only one row per band and that evaluation uses the final quantity band.

Steps:
1. Prepare CSV: `product_code, tier_min, tier_max, rate[, model_type, currency, effective_start, effective_end, reason_code, notes]`
2. Upload and map tier and rate fields.
   - Confirm numeric types and null handling for open-ended tiers.
   - Map `model_type` correctly if included.
3. Validate and apply updates.
   - The loader checks for overlaps, gaps, out-of-order rows, currency mismatches, and conflicting effective dates.
   - Choose whether to replace the entire tier table or merge changes by band.

Testing:
- Use quantity probes at band edges \(for example, 99, 100, 101\) to verify correct tier application.
- Validate both current and future-dated tables if effective dating is used.

---

## Limitations & unsupported use cases

- Only updates existing products; it does not create new products.
- Supports defined entities and formats; custom objects may be out of scope.
- Does not recalculate historical invoices. Changes affect new rating events from their effective dates.
- Maximum file size and row limits may apply. Large loads should be split into multiple batches.
- Cross-entity dependencies such as add-on bundles may require coordinated updates to avoid temporary inconsistencies.
- Timezone handling requires consistent datetime formats; mixed timezones in one file can cause boundary errors.
- Promotions with complex stacking across multiple discounts may require the promotions engine instead of the Standard Rate Loader.

The Standard Rate Loader streamlines rate maintenance for routine updates, promotions, and tier changes at scale.

