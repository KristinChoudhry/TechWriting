
# Standard Rate Loader: Common Use Cases

The Standard Rate Loader supports bulk rate management scenarios for pricing and billing teams. This article covers updating standard rates, applying promotions, and managing tiered pricing. These use cases improve accuracy and speed while reducing manual effort. Intended readers include Billing Administrators and Pricing Managers.

## Apply updated rates

1. Prepare CSV: `product_code`, `new_rate`, `currency`, `effective_start`.
2. Open Standard Rate Loader and create a new upload.
3. Upload the CSV and map fields.
4. Validate changes.
5. Click Apply Updates to confirm.

## Apply promotional rates

Run time‑bound discounts for targeted products.

### Schedule a promotion

1. Prepare CSV: `product_code`, `promo_rate`, `promo_start`, `promo_end`.
2. Upload and map fields, including dates.
3. Validate and set the schedule to the campaign start time.

## Manage tiered pricing

Adjust tiers for usage‑based or volume‑based models.

### Update pricing tiers

1. Prepare CSV: `product_code`, `tier_min`, `tier_max`, `rate`.
2. Upload and map tier and rate fields.
3. Validate and apply updates.

## Limitations & unsupported use cases

- Only updates existing products; it does not create new products.
- Supports defined entities and formats; custom objects may be out of scope.

The Standard Rate Loader streamlines rate maintenance for routine updates, promotions, and tier changes at scale.
