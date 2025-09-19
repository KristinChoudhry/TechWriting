# Configure Rate Uploads

This article explains how to configure the Rate Loader to apply bulk pricing updates from CSV files. You will prepare the data, upload it, map fields, validate, and apply updates.

---

## Prerequisites:

- **Role**: Billing Administrator or Pricing Manager
- **Access**: Product Catalog and Pricing Rules
- **Data file**: A UTF-8 CSV file with required columns

Additional prerequisites and environment readiness:
- **Price book access**: You can select and edit the intended price book or rate table environment (for example, Sandbox vs. Production).
- **Time zone alignment**: You know the loader’s default time zone and the time zone used in your CSV timestamps.
- **Currency enablement**: All currencies in the CSV are enabled and mapped to ISO 4217 codes (for example, `USD`, `EUR`).
- **Product readiness**: Target products exist, are active or activatable, and have valid `product_code` values.
- **Change control**: A rollback plan exists (for example, a baseline export of current rates and effective dates).
- **File limits**: Your file adheres to platform limits (row count, file size). Split files if needed.

CSV data requirements:
- **Required columns**: `product_code`, `rate`, `currency`, `effective_start`
- **Optional columns**: `effective_end`, `reason_code`, `notes`, `rounding_rule`, `price_book`, `region`, `segment`
- **Formats**:
  - `rate`: Decimal with up to 4 fractional digits; must be ≥ 0
  - `effective_start` and `effective_end`: `YYYY-MM-DD` or `YYYY-MM-DDTHH:MM:SSZ`
  - `currency`: ISO 4217 uppercase
- **Integrity**:
  - No overlapping effective date ranges for the same `product_code`/`currency`/`price_book`
  - If `effective_end` is provided, it must be greater than `effective_start`

---

## Configure a rate update upload

1. **Navigate** to `Pricing → Rate Management → Standard Rate Loader`.
2. **Click** Create New Upload.
3. **Upload your CSV file.**
   - Confirm encoding is UTF-8 without BOM.
   - Verify delimiter (`,` default) and text qualifier (`"`).
   - If using large files, consider compressed upload if supported.
4. **Map CSV columns to fields** (for example, `Product Code`, `Rate`, `Currency`, `Effective Start/End`).
   - Required field mappings:
     - `product_code` → Product Code
     - `rate` → Rate
     - `currency` → Currency
     - `effective_start` → Effective Start
   - Optional mappings:
     - `effective_end` → Effective End
     - `price_book` → Price Book (or choose in UI)
     - `region` / `segment` → Scoping attributes (if supported)
     - `reason_code` → Change Reason
     - `rounding_rule` → Rounding Rule
     - `notes` → Notes
   - Mapping tips:
     - Ensure date columns parse using the intended format and time zone.
     - Confirm numeric parsing for `rate` (no thousands separators, dot as decimal).
     - Standardize `product_code` case to match catalog settings.
5. **Select the run mode**: Preview (validate only) or Apply (execute now or schedule).
   - Preview performs all validations and produces a change set without persisting changes.
   - Apply executes changes immediately or at a scheduled time.
6. **Review the validation report** for errors and warnings.
   - Errors block execution (for example, missing required columns, invalid currency, unknown product).
   - Warnings highlight potential issues (for example, overlapping dates that will be resolved by end-dating).
   - Use filters to review by severity, column, and product.
   - Export the report to CSV for remediation workflows.
7. **Click Apply Updates** or set a schedule and confirm.
   - Choose the target price book/environment if not mapped per row.
   - Confirm conflict handling and effective date policy (see Configuration options).
   - Optionally notify stakeholders via email or webhook on completion.

Post-apply verification:
- Spot-check sample products with rate preview for boundary times (start minus 1 minute, start, end, end plus 1 minute).
- Export a diff report (before vs. after) using `reason_code` or upload ID.
- Create a rollback plan by preserving the validated input and the audit export.

---

## Configuration options

| Option | Description | Recommendation |
|---|---|---|
| Run Mode | Preview vs. Apply | Use Preview for first-time changes, large batches, or new mappings. For recurring jobs, keep a saved configuration and run Preview weekly before Apply. |
| Schedule | Future start time for apply | Choose off-hours aligned to the pricing engine’s time zone. Consider a 5–10 minute buffer before/after the hour to avoid batch peaks. |
| Conflict Handling | Overwrite or skip existing rates | Overwrite intentionally with audit. Prefer “insert and end-date current row” over hard overwrite to preserve history. Use Skip for exploratory loads. |
| Currency Enforcement | Restrict allowed currencies per product | Match price book settings. Reject rows where currency is not enabled for the product/region. |
| Effective Date Policy | Handling of missing effective dates | Require explicit start dates. Optionally auto-end-date the current row at `effective_start - 1 second` when inserting a new rate. |
| Scope Filters | Limit by price book, region, segment | Apply explicit scope to reduce accidental cross-book updates. |
| Rounding and Precision | Control rounding at write time | Use standard rounding (half up) to currency precision; override per row only with approved `rounding_rule`. |
| Change Reason | Tag changes for audit | Require `reason_code` for production loads (for example, `ANNUAL_REPRICE`, `PROMO_2025Q1`). |
| Duplicate Handling | Behavior for repeated rows within a file | Collapse exact duplicates or flag as warnings; reject conflicting duplicates. |
| File Partitioning | Split large loads into chunks | Use stable sort and deterministic chunking (for example, by product family) to ease retries. |
| Notifications | Completion and failure alerts | Enable email or webhook with summary metrics and a link to the audit. |

Additional option details:
- **Conflict Handling mechanics**:
  - Overwrite: Replace the existing row with identical effective range. Use sparingly.
  - Insert + End-date: Insert a new row at `effective_start` and end-date the previously active row. Preferred for auditability.
  - Skip: Ignore conflicting rows; report as warnings.
- **Effective dating safeguards**:
  - Enforce non-overlapping windows per `product_code`/`currency`/`price_book`.
  - Reject back-dated changes that would alter posted invoices unless explicitly allowed.

---

## Example configurations

- **Regional EU update**: Currency = `EUR`; target EU price book; schedule 02:00 local time.
  - Columns: `product_code, rate, currency, effective_start, price_book, region, reason_code`
  - Options: Run Mode = Preview, Conflict Handling = Insert + End-date, Currency Enforcement = On, Effective Date Policy = Require start
  - Tips: Validate VAT tax categories separately if regional books use different tax mappings.

- **Promotional discount**: Include promo start/end; set Conflict Handling = Overwrite.
  - Columns: `product_code, rate, currency, effective_start, effective_end, reason_code`
  - Options: Run Mode = Preview then Apply, Schedule = Campaign start - 5 minutes, Conflict Handling = Insert + End-date (preferred), Change Reason = `PROMO_SPRING_2025`
  - Tips: If overlapping with standard rates, confirm the engine selects the lower eligible rate or configure promo price book precedence.

- **Multi-currency refresh**: Update USD and GBP simultaneously.
  - Columns: `product_code, rate, currency, effective_start, price_book`
  - Options: Currency Enforcement = On, Duplicate Handling = Collapse duplicates, Notifications = On
  - Tips: Verify rounding to 2 decimals for both currencies and confirm FX policy alignment.

- **Contract-aligned reprice**: Align standard rates with new vendor costs.
  - Columns: `product_code, rate, currency, effective_start, reason_code, notes`
  - Options: Conflict Handling = Insert + End-date, Effective Date Policy = Require start, Schedule = Off-hours
  - Tips: Attach `reason_code = COST_PASS_THROUGH` for downstream reporting.

---

## Troubleshooting

- **Upload fails**
  - Check CSV encoding (UTF-8), delimiter, and presence of required columns.
  - Validate data types: numeric `rate`, ISO currency codes, parsable dates.
  - Look for unescaped quotes or line breaks inside fields; re-export with proper quoting.
  - Verify file size and row count limits; split the file if necessary.

- **No changes applied**
  - Confirm field mappings saved correctly; re-open mapping screen to verify.
  - Ensure products are Active or in a state that allows rate updates.
  - Check Conflict Handling: if set to Skip, rows may be ignored on conflicts.
  - Review validation report filters; errors might be hidden by current filter.

- **Wrong price book**
  - Verify selected price book/environment before Apply.
  - If `price_book` is included in the CSV, ensure values match exact codes.
  - Use Scope Filters and lock to a single book when in doubt.

- **Overlapping effective dates**
  - Adjust `effective_end` on existing rows or choose Insert + End-date.
  - Use Preview to detect overlaps before Apply.

- **Time zone mismatches**
  - Align CSV datetimes to the loader’s expected time zone or include offsets.
  - Test boundary minutes around activation/deactivation times.

- **Unexpected rounding**
  - Confirm currency precision and `rounding_rule` application.
  - Remove thousands separators and ensure dot decimal format.

- **Performance and timeouts**
  - Partition large uploads by product family or currency.
  - Schedule off-hours and enable retries for failed chunks.

- **Audit and rollback**
  - Use `reason_code` to find and export applied changes.
  - Roll back by reloading the previous rate set with a new `effective_start` that supersedes the change.

These steps and options cover the most common setups for safe, repeatable pricing updates, while providing the controls and safeguards needed for production-grade operations.

