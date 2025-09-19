# Configure Rate Uploads

This article explains how to configure the Rate Loader to apply bulk pricing updates from CSV files. You will prepare the data, upload it, map fields, validate, and apply updates.

Prerequisites:
- Role: Billing Administrator or Pricing Manager
- Access to Product Catalog and Pricing Rules
- A UTF‑8 CSV file with required columns

## Configure a rate update upload

1. Navigate to Pricing → Rate Management → Standard Rate Loader.
2. Click Create New Upload.
3. Upload your CSV file.
4. Map CSV columns to fields (e.g., Product Code, Rate, Currency, Effective Start/End).
5. Select the run mode: Preview (validate only) or Apply (execute now or schedule).
6. Review the validation report for errors and warnings.
7. Click Apply Updates or set a schedule and confirm.

## Configuration options

| Option                 | Description                                   | Recommendation                      |
|-----------------------|-----------------------------------------------|-------------------------------------|
| Run Mode              | Preview vs. Apply                             | Use Preview for first‑time changes  |
| Schedule              | Future start time for apply                   | Choose off‑hours                    |
| Conflict Handling     | Overwrite or skip existing rates              | Overwrite intentionally with audit  |
| Currency Enforcement  | Restrict allowed currencies per product       | Match price book settings           |
| Effective Date Policy | Handling of missing effective dates           | Require explicit start dates        |

## Example configurations

- Regional EU update: Currency=EUR; target EU price book; schedule 02:00 local time.
- Promotional discount: Include promo start/end; set Conflict Handling=Overwrite.

## Troubleshooting

- Upload fails: Ensure CSV is UTF‑8, required columns exist, and data types are valid.
- No changes applied: Verify field mappings and confirm products are active.
- Wrong price book: Confirm the selected book/environment matches your intent.

These steps and options cover the most common setups for safe, repeatable pricing updates.
