# Tutorial: Usage Based Billing

This tutorial guides you through configuring a usage product, creating a data collector, and mapping incoming usage fields for usage‑based billing. Time to complete: ~45 minutes.

Before you begin:
- Admin access to the platform
- Understanding of your source usage fields
- Sample CSV or an API endpoint with usage events

Time to complete: Approximately 30 minutes

## Create a usage product

### Define product details

1. Navigate to Product Catalog → New Product.
2. Enter a clear name (e.g., “API Calls”).
3. Set the type to Usage and save.

### Specify unit of measure

1. In product settings, set the unit (e.g., “API Call”).
2. Confirm the unit displays properly on invoices.

## Configure a data collector

### Select collector type

1. Open Integrations → Collectors → New.
2. Choose CSV Upload, S3 Watcher, or API Polling.

### Define source and schedule

1. Enter source details (file path, bucket, or API URL).
2. Set frequency (e.g., every 15 minutes).
3. Provide credentials if required and save.

## Establish usage data mappings

### Select data source

1. Go to Usage Mappings → New.
2. Choose your collector as the source.

### Map fields

1. Map `quantity` → Usage Quantity.
2. Map `account_id` → Account Identifier.
3. Map `service_code` → Product Code.
4. Map `event_datetime` → Usage Date/Time.
5. Save the mapping.

## Troubleshooting

- Error: Unmapped Product Code
  - Cause: Product code missing in catalog
  - Fix: Create the product or correct mapping

- Records processed but not billed
  - Cause: Missing subscription/pricing
  - Fix: Ensure active subscription and price book
