# Bulk Create Customers

This guide explains how to bulk‑create customer accounts from a CSV file. Importing customers accelerates onboarding and reduces manual entry errors.

Prerequisites:
- Permission: Customer Import or Administrator

## Import customers

1. Go to Data Management → Imports → Customers.
2. Click Create Import and choose File Type: CSV.
3. Upload the CSV file.
4. Map source columns (e.g., `external_id`, `name`, `email`, `billing_country`) to target fields.
5. Validate the preview and address any errors.
6. Click Submit Import to run the job.
7. Monitor status and download the results report.

## Use case: Migrate 1,000 customers from legacy CRM

- Map `external_id` → Account External ID for deduplication.
- Default Country to “US” when blank via mapping rule.
- Rerun with “Update if exists” to correct rejected rows.

## Troubleshooting

- Invalid email errors: Fix formatting in the CSV and re‑upload.
- Duplicates detected: Enable “Update if exists” and key on External ID.
- Slow or stalled job: Check file size limits and run during off‑peak windows.

After import, link subscriptions and assign price books as needed.
