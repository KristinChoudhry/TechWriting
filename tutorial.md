
# Tutorial: Set Up Basic Usage Billing 

This tutorial guides you through the process of configuring the core components for usage-based services. By following these steps, you will learn how to define a billable usage product, set up a data collector for usage events, and map incoming data fields to entities. This is useful when you want to accurately charge customers based on their consumption of services like API calls, data storage, or transactions.

- Time to complete: Approximately 60–90 minutes.

---

## Before you begin

Ensure you have the following prerequisites in place:

- Software/Tools:
  - Access to your instance with administrator or configurator rights.
  - Credentials for any external data source you will connect (for example, API keys, S3 IAM role, or secure file share access).
- Knowledge:
  - Understanding of your company’s usage data sources and the fields they contain (for example, customer/account identifiers, product/service codes, quantities, and timestamps).
  - Familiarity with the UI structure for product catalog, data collection, and mappings.
- Files/Resources:
  - A sample usage data file (CSV) if configuring a file-based collector.
  - Details of an API endpoint or storage bucket that provides usage events if configuring an automated collector.

---

## Create a Usage Product

In this section, you will define the product that will be associated with the collected usage data.

### Define Product Details

1. Navigate to Products > Product Management > Products.
2. Select New to create a new product.
3. Enter a clear, customer-facing name, for example, “API Transactions” or “Data Storage (GB)”.
4. Set the product type/category appropriate for metered billing (for example, Usage).
5. Save the product to persist the base configuration.

Note: Ensure the product type supports metered/usage rating. This decision influences downstream pricing and reporting.

### Specify Unit of Measure

1. Open the product you created and locate the Unit of Measure field.
2. Define the unit by which usage will be measured, such as “API Call”, “GB”, or “User Session”.
3. If available, set precision/rounding rules to control how quantities are rated.

Tip: Select a unit name that matches how your source system counts events (for example, if the source logs bytes, decide whether to convert to MB/GB in mapping or rating).

### Configure Pricing and Rating

1. Go to the product’s Pricing or Rating section.
2. Choose a Rating Method suitable for usage (for example, flat per-unit rate, tiered, or volume).
3. Define the price or tiers:
   - Flat per-unit example: $0.002 per API Call.
   - Tiered example: 0–1M calls at $0.002 per call; 1M+ calls at $0.0015 per call.
4. If needed, associate the product with a price book or contract-level rates to support customer-specific pricing.
5. Save your pricing configuration.

Tip: If you expect large usage volumes, prefer tiered pricing for progressive discounts; use volume pricing to apply a retroactive rate to the entire quantity once a threshold is reached.

---

## Configure a Data Collector

Here, you will set up the mechanism to bring usage data into the platform.

### Select Collector Type

1. Navigate to the data collection or integration management area in your instance.
2. Choose Add New Collector.
3. Select the collector type based on your source:
   - API Polling for REST endpoints that expose usage events.
   - S3/Blob Storage Watcher for batch files landed in buckets/containers.
   - Manual CSV Upload for periodic backfills or initial testing.

Expected outcome: The configuration screen for the selected collector type appears.

### Define Collector Source and Schedule

1. Enter source details, for example:
   - API Polling: Base URL, resource path, query parameters (date cursors, pagination).
   - S3 Watcher: Bucket name, folder/prefix, file pattern (for example, usage_*.csv).
   - CSV Upload: File format profile and column delimiter/headers.
2. Configure authentication:
   - API keys, OAuth client credentials, or custom headers for APIs.
   - IAM role, access key/secret, or workload identity for S3/cloud storage.
3. Set the collection frequency:
   - For near-real-time: Every 5–15 minutes.
   - For batch: Hourly or daily at a fixed time aligned to source publication.
4. Define fault-handling/offsets:
   - Enable retry with backoff on transient failures.
   - Configure checkpointing (for example, last processed timestamp or file marker).
5. Save the collector configuration.

Expected outcome: The new collector appears in the list with status Idle, Scheduled, or Running (depending on type).

---

## Establish Usage Data Mappings

This section covers linking fields from your incoming usage records to the appropriate entities.

### Select Data Source for Mapping

1. Navigate to the Usage Mapping or Data Transformation area.
2. Create a new mapping rule and associate it with the data collector you configured (for example, “API Usage Collector”).

### Map Source Fields to Target Fields

1. Identify the source field that represents the quantity consumed (for example, numberOfUnits, callsCount, bytesUsed).
2. Map this source quantity field to the “Usage Quantity” target field.
3. Map entity identifiers:
   - Customer: Map the source account identifier (for example, account_id) to the “Account Identifier” field that matches an existing account.
   - Product: Map the product/service code (for example, service_code or sku) to the “Product Code” field that matches your Usage Product.
4. Map temporal and contextual attributes:
   - Timestamp: Map event_datetime to “Usage Date/Time”.
   - Optional: Region, planId, and any rating attributes used by pricing rules.
5. Apply transformations as needed:
   - Unit conversions: bytes → GB, ms → seconds.
   - Normalization: trim spaces, uppercase product codes.
   - Derivations: concatenate fields to build composite keys (for example, region + code).
6. Save the mapping rules.

Tip: Validate a small sample first—misaligned identifiers are the most common cause of orphaned usage.

---

## Test and Validate Usage Processing

In this section, you will verify end-to-end ingestion and rating on a controlled sample.

### Load a Sample Batch

1. Prepare a small dataset (10–100 rows) that includes known accounts, product codes, quantities, and timestamps.
2. Trigger the collector:
   - For API/S3: Place the sample where the collector can retrieve it; optionally force a run.
   - For CSV: Use the manual upload function with your sample file.
3. Monitor the collector status until the batch is ingested.

Expected outcome: The system acknowledges files/events as processed and moves them into mediation.

### Review Mediation and Rating Results

1. Open the usage processing/mediation monitor to review accepted and rejected records.
2. Investigate rejected items; note error messages (for example, “Unmapped Product Code”, “Account not found”).
3. For accepted records, confirm:
   - Correct account linkage.
   - Correct product association.
   - Correct quantity and unit conversions.
   - Correct timestamps/time zone handling.
4. Open rated charges or a test invoice preview for the affected account to verify amounts align with pricing rules.

Tip: Test edge cases (zero quantities, boundary timestamps, tier breakpoints) to confirm pricing logic.

---

## Operationalize Scheduling and Controls

Once validated, set up long-running operations and safeguards.

### Finalize Schedules and Windows

1. Align collection schedules with source publication times to reduce empty runs.
2. Define maintenance windows to pause ingestion during system updates.
3. Stagger high-volume collectors to avoid rating contention spikes.

### Set Alerts and Observability

1. Configure alerts on:
   - Collector failures or excessive retries.
   - Sudden drops or spikes in daily record counts.
   - High rejection rates in mediation.
2. Expose dashboards:
   - Volume trends by product/account.
   - Rejection categories and top offenders.
   - Processing latency from ingestion to rating.

### Establish Data Governance

1. Define retention for raw files and mediated records according to compliance.
2. Implement deduplication safeguards (idempotency keys or file manifest hashing).
3. Document field contracts with source teams and version them.

---

## Troubleshooting

Use this section to address common issues users may encounter while performing the task.

- Issue: Collected usage data isn’t associated with any accounts.
  - Cause: The account identifier in source data doesn’t match an existing Account Identifier.
  - Resolution:
    1. Verify the mapping points to the correct source field (for example, account_id).
    2. Confirm the values match existing accounts; correct casing and remove whitespace.
    3. Reprocess the rejected records after fixing mappings or account master data.

- Error Message: “Unmapped Product Code: <value>”
  - Cause: The usage data contains a product code that doesn’t exist or isn’t configured as a Usage Product.
  - Resolution:
    1. Create or update the product with the missing code.
    2. Ensure the product type supports usage rating and has pricing defined.
    3. If the source field is wrong, update the mapping to use the correct product code field.

- Issue: Quantities look incorrect after rating (too large or too small).
  - Cause: Missing or incorrect unit conversion in mappings.
  - Resolution:
    1. Confirm the source unit (for example, bytes) and desired target unit (for example, GB).
    2. Apply a conversion transform in the mapping (for example, divide by 1024^3).
    3. Re-rate a small sample to confirm correctness.

- Issue: Late-arriving events missed the intended billing period.
  - Cause: Ingestion lag or timestamp timezone misalignment.
  - Resolution:
    1. Normalize all timestamps to UTC during mapping.
    2. Configure rating to respect event time for period assignment where supported.
    3. If supported, run a catch-up rating job for late records.

---

## Next steps

After completing this tutorial, you might want to explore these related topics:

- Configure tiered or volume pricing for usage products to support advanced discount structures.
- Set up aggregation rules for daily or monthly usage summaries to optimize invoice readability.
- Explore advanced collector settings (for example, API pagination, rate limiting, checksum validation).
- Implement customer-specific price books and contracts for negotiated rates.

---

In this tutorial, you learned how to:

- Define a Usage Product with a specific unit of measure like “API Call” or “GB”.
- Set up a Data Collector to retrieve usage events from a source like an API endpoint, cloud storage, or a CSV file.
- Create Mappings to link incoming data fields (for example, account_id, service_code, quantity, event_datetime) to entities and attributes.
- Validate end-to-end processing and put operational controls in place.

You should now be able to configure the foundational elements for processing and billing based on customer usage within the platform.
