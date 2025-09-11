# Troubleshoot the Standard Rate Loader

Use this guide to resolve common issues encountered when updating rates in bulk.

## Issue: CSV file upload fails

- Symptoms: Error during upload; no data imported.
- Possible causes: Wrong file format; missing required columns; invalid data types.

### Steps to resolve

1. Verify that the file is CSV (UTF‑8).
2. Confirm required columns (e.g., product code, rate, currency) are present.
3. Ensure numeric fields contain valid numbers.
4. Re‑upload and review the validation report.

- Resolution: The file uploads successfully and data is queued for processing.

## Issue: Rates do not update

- Symptoms: System rates do not match CSV values after job completion.
- Possible causes: Incorrect field mapping; caching; scheduled job not executed.

### Steps to resolve

1. Review field mappings for accuracy.
2. Clear or bypass caches per environment policy.
3. Verify the scheduled update executed and succeeded.

- Resolution: Updated rates appear as expected in the catalog.
