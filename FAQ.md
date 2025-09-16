# Developer FAQ — Usage Billing and Rating

This article follows the provided FAQ template and is targeted at engineers, integration developers, solution architects, and SREs implementing high-scale usage ingestion, mediation, pricing, and rating. It assumes familiarity with event-driven data pipelines, CI/CD, and production operations.

---

## Table of Contents

- Overview
- Data Ingestion and Event Design
- Mediation, Mapping, and Transformations
- Pricing, Rating, and Discounts
- Scale, Performance, and Throughput
- Versioning, Testing, and Deployment
- Security, Compliance, and Governance
- Troubleshooting and Operational Playbook
- Runbooks and Checklists
- Glossary
- References and Next Steps

---

## Overview

- **Purpose**: Provide a deep, developer-focused FAQ for building resilient, auditable, and scalable usage billing pipelines.
- **Audience**: Backend/Data Engineers, Integration Developers, Pricing Engineers, DevOps/SRE, and Technical Admins.
- **Assumptions**:
  - You can configure products, collectors, and mappings.
  - You use infrastructure-as-code and CI/CD for configuration changes.
  - You understand event-time vs processing-time semantics and idempotency.

---

## Data Ingestion and Event Design

### What is the recommended event schema for reliable, idempotent ingestion?
- **Required fields**:
  - `event_id`: deterministic and globally unique.
  - `account_id`: resolvable to a Billing Account Identifier.
  - `product_code`: resolvable to a configured Usage Product or pricing group.
  - `quantity`: non-negative numeric; prefer base units.
  - `event_time`: ISO-8601, UTC (e.g., `2025-09-15T12:34:56.789Z`).
- **Recommended fields**:
  - `unit`: base unit label like `BYTE`, `CALL`, `MS`.
  - `region`, `planId`, `endpoint_tier`: contextual attributes for pricing rules.
  - `ingest_time`: ingestion timestamp for observability.
  - `source_system`, `file_name`, `line_no`: provenance for audit/debug.
- **Keying**:
  - Build `event_id` from immutable attributes: `(source_system, account_id, product_code, event_time, sequence, crc(payload))`.

Example (NDJSON):
```json
{"event_id":"srcA|ACC-42|API-TXN|2025-09-15T12:34:56.789Z|000042|f1a9","account_id":"ACC-42","product_code":"API-TXN","quantity":1,"unit":"CALL","event_time":"2025-09-15T12:34:56.789Z","region":"us-east-1","attributes":{"planId":"PRO"}}
```

Deterministic ID (bash-like pseudocode):
```bash
event_id="$(printf '%s' "srcA|$account_id|$product_code|$event_time|$sequence|$payload_crc" | sha256sum | cut -d' ' -f1)"
```

### Which file formats are best for batch ingestion?
- **CSV**: fastest and compact; include a header, use UTF-8, quote fields containing delimiters, avoid locale-specific decimal formats.
- **JSON Lines (NDJSON)**: flexible for sparse or nested attributes; one event per line.
- **Parquet**: columnar and efficient for very large volumes; ideal when using a data lake and ETL before collection.

> Tip: Normalize all timestamps to UTC ISO-8601. Avoid local times and ambiguous formats.

### Push vs. Pull ingestion — how should I choose?
- **Pull** (API polling, S3/Blob watcher):
  - Pros: control, easy backfills, deterministic checkpoints.
  - Cons: latency tied to schedule; polling overhead.
- **Push** (webhooks, event bus, streaming):
  - Pros: low latency, natural backpressure via queues.
  - Cons: harder replay; must implement durable sinks.
- **Hybrid**:
  - Accept pushes into durable storage (queue/object store), then run scheduled collectors for mediation.

**Choose based on SLAs**:
- If RTO/RPO requires \(\le 5\) minutes, use push or short-interval pull with backpressure and idempotent replays.

### How do I ensure idempotent ingestion and safe reprocessing?
- **Deterministic IDs**: derive `event_id` from immutable attributes.
- **Checkpointing**: track last processed watermark (timestamp, file marker).
- **Dedup**: implement upsert keyed on `event_id`; maintain a short TTL cache for near-duplicates.
- **Poison handling**: quarantine files after N retries; alert with checksum and location.
- **Replay plan**: re-run by partition (`event_date`) with same IDs; measure deltas via control tables.

---

## Mediation, Mapping, and Transformations

### How do I map source fields to platform entities reliably?
- **Core mappings**:
  - `account_id` → Account Identifier.
  - `product_code` → Usage Product Code or rating group.
  - `quantity` → Usage Quantity (ensure numeric type).
  - `event_time` → Usage Date/Time (UTC).
- **Context attributes**:
  - `region`, `planId`, `endpoint_tier`, `time_bucket`.
- **Normalization**:
  - Codes: `upper()`, `strip()`, `replace('-', '')`.
  - Units: bytes→GB, ms→s.
  - Timestamps: parse as UTC; reject ambiguous inputs.

Example transform (Python-like):
```python
row["product_code"] = row["product_code"].strip().upper()
row["quantity_gb"] = float(row["bytes_used"]) / (1024 ** 3)
row["event_time"] = parse_iso_utc(row["event_time"])
row["time_bucket"] = "PEAK" if 14 <= row["event_time"].hour < 22 else "OFFPEAK"
```

### How should I handle multi-dimensional usage (e.g., call count plus data size)?
- Keep a primary `quantity` for the rated unit (e.g., `CALL`).
- Add secondary attributes for modifiers (e.g., `data_size_gb`, `endpoint_class`).
- Configure rating rules that reference these attributes (selectors, multipliers, or separate product components).

### What about timezone and billing period assignment?
- Store everything in UTC.
- Assign billing periods using `event_time` in UTC.
- Only convert to tenant-local time at reporting if required; document cutover rules (e.g., midnight local vs UTC).

### How do I validate mappings before production?
- Create a small golden dataset with:
  - Nominal events, zero quantities, tier boundaries, cross-timezones, late arrivals.
- Run through mediation; assert:
  - No unmapped accounts/products.
  - Correct unit conversions.
  - Accurate timestamps and period assignment.
- Diff rated outputs against expected values; gate deploy on test pass.

---

## Pricing, Rating, and Discounts

### Tiered vs. Volume pricing — when to use which?
- **Tiered (progressive)**: Each bracket has its own rate; total price is the sum across brackets.
- **Volume (block)**: Determine a single effective rate from the final bracket and apply to all units.
- Use **tiered** for progressive discounts and **volume** for “all units at a rate” simplicity.

### How do I pass dynamic attributes into rating?
- Include fields such as `region`, `planId`, `endpoint_tier`, `time_bucket`.
- Map them to rating attributes.
- Reference attributes in rate conditions:
  - Example: apply EU regional rate when `region == "EU-WEST"` and `planId == "ENTERPRISE"`.

### Implementing free tiers and overage
- **Option A**: Tier 1 at rate \$0 up to \(N\) units; subsequent tiers at normal rates.
- **Option B**: Maintain counters; zero-rate until counter exceeds \(N\), then apply overage rates.
- Ensure idempotent counter updates and consistent event ordering for correctness.

### Can I support burst pricing or time-of-day rates?
- Derive `time_bucket` (e.g., `PEAK`/`OFFPEAK`) and configure separate rate tables per bucket.
- For bursts, compute `rolling_window_qps` at mediation; use `burst_tier` attribute to select surge rates.

### Handling minimum commitments and prepaid credits
- Track commitment balance by period.
- Apply rating to draw down credits first; spill to overage products afterward.
- Ensure proration logic for mid-period plan changes.

---

## Scale, Performance, and Throughput

### What throughput can I expect and how do I plan capacity?
- **Partitioning**: by `event_date`, `account_id`, or `product_code` to parallelize ingestion and rating.
- **Batch sizing**: 10k–100k records per batch is a good starting range.
- **Avoid**:
  - Per-event remote lookups during mediation.
  - Highly variable schemas within a single batch.
- **Levers**:
  - Pre-aggregate where business rules allow (daily sums).
  - Normalize units pre-rating to avoid heavy transforms at runtime.

Back-of-envelope:
- If peak \(= 200\) million events/day, target \(\approx 2{,}500\) events/sec sustained with headroom; schedule batches to keep < 70% utilization to allow retries/backfills.

### How do I handle late-arriving data?
- Use event-time semantics.
- Define a lateness window (e.g., T+3 days).
- Schedule catch-up rating jobs and invoice adjustments where necessary.
- Keep mediation idempotent to support safe reprocessing.

### What about extremely large files?
- Prefer chunked ingestion or split by partition before upload.
- Validate schema and headers in a pre-check step; fail fast with clear diagnostics.
- Use checksum validation to detect truncation or corruption.

---

## Versioning, Testing, and Deployment

### How do I version mappings and price books safely?
- **Mappings**: `usage_mapping_vN` with semantic versioning; only add fields when possible.
- **Rates**: Version with effective dates `[start, end)`; never edit past-effective rates—create a new version.
- **Rollout**: Canary with a subset of accounts; compare rated totals before global cutover.

### Recommended testing strategy
- Golden test suites covering:
  - Nominal, zero, boundary tiers, multi-attribute selection, cross-timezone, late arrivals, and high volume.
- Automated assertions:
  - Rated totals, per-tier allocations, taxes, and rounding/precision.
- Shadow mode:
  - Run new mapping/rating in parallel; diff results against production for a defined window.

### CI/CD for configuration
- Store mappings/pricing as code (YAML/JSON) in VCS.
- Require code review and automated tests before deployment.
- Tag releases and maintain a changelog referencing incident IDs and JIRA tickets.

---

## Security, Compliance, and Governance

### How do I protect sensitive data in usage events?
- **Do**:
  - Use surrogate keys; keep PII out of events.
  - Hash or tokenize any unavoidable sensitive identifiers.
  - Encrypt in transit and at rest; restrict access via least privilege.
- **Don’t**:
  - Embed secrets (API keys, tokens); pass references instead.
  - Log full payloads without redaction.

### Data residency and regional pricing
- Include `region` as a first-class attribute.
- Maintain regional collectors/storage; process in-region to meet residency rules.
- Map `region` to regional price books and tax treatments.

### Auditing and controls
- Maintain control tables:
  - Daily counts, sums, min/max timestamps, rejection counts.
- Reconciliation:
  - Compare source vs processed metrics; alert on gaps > threshold.
- Retention:
  - Define retention for raw, mediated, and rated records aligned to policy.

---

## Troubleshooting and Operational Playbook

### Records not linking to accounts
- **Causes**: mismatched `account_id`, casing/whitespace, wrong mapping column.
- **Fix**:
  1. Inspect rejects with actual values.
  2. Normalize: `trim()`, `upper()`.
  3. Confirm account exists and identifiers match.
  4. Reprocess after correction.

### “Unmapped Product Code: <value>”
- **Causes**: missing product config; code drift between systems.
- **Fix**:
  1. Create/align Usage Product to source code.
  2. Normalize codes in mapping.
  3. Add regression tests for product code contracts.

### Quantities off by 1024×
- **Cause**: missing bytes→GB conversion.
- **Fix**:
  - Apply \( \text{GB} = \frac{\text{bytes}}{1024^3} \).
  - Align units and pricing tiers after conversion.

### Late events posted to wrong period
- **Cause**: local-time parsing or processing-time assignment.
- **Fix**:
  1. Normalize timestamps to UTC.
  2. Use event-time for period selection.
  3. Run catch-up rating and adjust invoices.

### Collector stuck retrying a bad file
- **Implement poison queue**:
  - Exponential backoff up to \(N\) attempts.
  - Quarantine after \(N\) with checksum and path.
  - Alert with remediation steps.

### High rejection rates after a deploy
- **Playbook**:
  1. Freeze further deploys; enable shadow rollback.
  2. Identify top rejection reasons and fields.
  3. Hotfix mapping/rates; re-run golden tests.
  4. Reprocess rejects; monitor deltas in control tables.

### How do I audit daily completeness?
- **Metrics**:
  - Event counts, quantity sums, min/max `event_time` per partition.
- **SQL sketch**:
```sql
SELECT d.event_date,
       s.source_events,
       p.processed_events,
       (s.source_events - p.processed_events) AS gap
FROM source_counts s
JOIN processed_counts p USING (event_date)
JOIN calendar d ON d.event_date = s.event_date
WHERE gap <> 0 OR p.processed_events < s.source_events * 0.99;
```

---

## Runbooks and Checklists

### Daily ops checklist
- [ ] Collectors healthy; no stuck retries or quarantines growing.
- [ ] Mediation rejection rate < 1% (or agreed SLO).
- [ ] Control table parity within threshold (counts and sums).
- [ ] Rating latency within SLO; backlog < threshold.
- [ ] Alerts reviewed; incidents triaged.

### Release checklist (mapping/rates)
- [ ] Change reviewed; effective dates correct.
- [ ] Golden tests pass; shadow diff within tolerance.
- [ ] Backout plan documented; toggle/feature flag ready.
- [ ] Comms prepared for business stakeholders.

### Backfill checklist
- [ ] Partition plan approved; idempotent IDs verified.
- [ ] Notifications muted to avoid customer artifacts.
- [ ] Post-partition reconciliation complete; discrepancies resolved.
- [ ] Invoices re-rated where applicable.

---

## Glossary

- **Event-time**: The timestamp when the usage occurred; used for billing period assignment.
- **Processing-time**: The time the system ingests/processes the event.
- **Mediation**: Transformation/normalization stage mapping raw events to rating-ready records.
- **Idempotency**: Property that reprocessing the same input yields the same result without duplication.
- **Tiered pricing**: Progressive bracket pricing where each tier has its own rate.
- **Volume pricing**: Single rate applied to all units based on total volume bracket.

---

## References and Next Steps

- Configure and version Usage Products and price books with explicit effective dates.
- Implement regional collectors and attribute-driven pricing for geo-specific rates.
- Build automated golden tests and shadow runs into CI/CD.
- Expand observability: ingestion lag, rejection taxonomy, rating throughput, and end-to-end latency dashboards.
- Document field contracts with source systems; enforce with schema validation at ingress.

---

## Quick Copy Snippets

Event schema (JSON Lines):
```json
{"event_id":"<sha256>","account_id":"ACC-123","product_code":"API-TXN","quantity":1,"unit":"CALL","event_time":"2025-09-15T12:34:56Z","region":"us-east-1","attributes":{"planId":"PRO","endpoint_tier":"A"}}
```

Bytes to GB conversion (SQL):
```sql
SELECT bytes_used / POWER(1024, 3) AS quantity_gb FROM events;
```

Deterministic ID (Python):
```python
import hashlib

def event_id(source, account_id, product_code, event_time, sequence, payload_crc):
    raw = f"{source}|{account_id}|{product_code}|{event_time}|{sequence}|{payload_crc}".encode("utf-8")
    return hashlib.sha256(raw).hexdigest()
```

---

By following this FAQ, you can design and operate a robust, auditable, and scalable usage billing pipeline with clear operational controls, strong security posture, and predictable pricing outcomes. Keep mappings, price books, and tests versioned; use event-time semantics; and automate reconciliation to catch issues early.
```
