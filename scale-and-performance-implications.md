# Dynamics Connector: Scale and Performance Implications

This guide provides engineering‑level guidance on scale, throughput, and tuning strategies for the Dynamics Connector. It covers observed limits, configuration levers, environment dependencies, and common anti‑patterns. Use it to plan capacity, estimate run times, and avoid operational bottlenecks in production.

> Note: Replace placeholder metrics with your validated values for your environment. Performance varies by Dynamics version, API type, network, and data shape.

---

## At‑a‑glance summary

- **Throughput (read)**: Approximately \(x\)-\(y\) records/second per worker under typical load, assuming average row width and moderate server latency.
- **Throughput (write)**: Approximately \(a\)-\(b\) records/second per worker, dependent on batch size, validation rules, and server throttling.
- **Recommended batch size**: \(500\)-\(2{,}000\) records per request for reads; \(100\)–\(500\) for writes, tuned to payload size limits.
- **Parallelism**: Start with \(p=2\)-\(4\) concurrent workers per entity; increase cautiously while monitoring API throttling and DB contention.
- **Schedule frequency**: Do not schedule more frequently than the typical full run time for the same scope. Leave a \(20\%\) buffer for retries and backoff.
- **Preferred strategy**: Incremental syncs using checkpoints and server‑side filters. Avoid full table scans during business hours.

---

## 1. Architecture and performance factors

The Dynamics Connector typically interacts via the Dataverse Web API or the Dynamics 365 Organization Service. Performance depends on:

- **API semantics and limits**
  - Request size caps (e.g., 16 MB payloads), server paging, and max page size controls.
  - Throttling policies that enforce per‑user and per‑application limits.
- **Data shape**
  - Average record size (field count and large text/BLOBs), column selectivity, and relationship depth.
- **Query complexity**
  - Filter selectivity, joins/expands, and server‑side vs client‑side filtering.
- **Network**
  - Round‑trip latency \(L\) and bandwidth \(B\), especially across regions or VPNs.
- **Concurrency model**
  - Number of parallel workers, batch sizes, retry policies, and idempotence of operations.
- **Environment load**
  - Background workflows, plug‑ins, and business rules executed on writes.

---

## 2. Throughput benchmarks

> Benchmarks are illustrative. Validate with your data in a staging environment that mirrors production latency and throttling.

- **Read throughput (server‑side filtered, average row width ≤ 4 KB)**
  - Single worker: \(x\)–\(y\) records/s
  - Four workers: approximately \(3x\)–\(3.5y\) records/s due to overhead and throttling
- **Write throughput (upsert with validation, batch size 200)**
  - Single worker: \(a\)–\(b\) records/s
  - Four workers: \(2.5a\)–\(3b\) records/s (diminishing returns from contention and throttling)
- **Latency sensitivity**
  - Effective throughput \(\approx \frac{\text{batch\_size}}{\text{RTT} + \text{server\_time}}\).
  - Increase batch size up to payload limits to amortize RTT, but monitor error rates and memory.

---

## 3. Scheduling and run‑time estimation

- **Do not overlap identical scopes.** Schedule frequency should be ≥ typical run time for that scope, with a buffer.
  - Example: If a full run takes 45 minutes, schedule at ≥ 55 minutes to allow for retries.
- **Prefer off‑peak windows** for full or heavy syncs to minimize contention with interactive users.
- **Estimate run time**
  - For \(N\) records and observed throughput \(T\) records/s across \(P\) workers:
    \[
    \text{ETA} \approx \frac{N}{T}
    \]
    where \(T\) is the measured aggregate throughput for that configuration.
- **Stagger entity syncs** to avoid bursty cross‑entity contention and API throttling.

---

## 4. Incremental vs. full sync

- **Always default to incremental.**
  - Use server‑side filters on `modifiedon`/`version` columns or change tracking if available.
  - Maintain durable checkpoints (high‑water marks) to resume reliably after failures.
- **When to run full syncs**
  - Schema changes that affect filtering, backfills, or data quality corrections.
  - Schedule during off‑hours and consider entity sharding by key ranges to parallelize safely.

---

## 5. Batch sizing and pagination

- **Reads**
  - Set page size to a moderate value (e.g., 1,000) to balance RTT and memory.
  - Avoid `expand` on large collections; fetch related entities separately if expansions inflate payloads.
- **Writes**
  - Use batch operations (e.g., 100-500 items) to amortize overhead.
  - Respect payload limits and validation costs; include only necessary fields.
- **Adaptive tuning**
  - Start small, observe error rates and latency, then scale up until you hit:
    - Elevated 429/503 responses (throttling)
    - Increased deadlocks/timeouts
    - Memory pressure or GC pauses on the client

---

## 6. Concurrency and parallelism

- **Baseline**: \(p=2\)-\(4\) concurrent workers per entity.
- **Scale cautiously**
  - Increase workers one step at a time; monitor success rate and latency.
  - Separate hot entities (e.g., `account`, `contact`) onto distinct worker pools.
- **Idempotence**
  - Make writes idempotent using upsert semantics or deterministic keys to allow safe retries.
- **Lock contention**
  - Avoid wide, frequent updates on heavily referenced entities during business hours.

---

## 7. Throttling, retries, and backoff

- **Detect throttling**
  - Handle HTTP 429/503 with `Retry-After` headers where present.
- **Use exponential backoff with jitter**
  - Backoff: \(t = \min(t_{\max}, 2^{k} \cdot t_{0} + \text{jitter})\)
- **Circuit breaking**
  - If sustained throttling occurs, reduce concurrency automatically and extend schedule windows.
- **Rate budgeting**
  - If Dynamics provides limits per app/user, partition credentials or schedule to stay within quotas.

---

## 8. Query design and anti‑patterns

- **Do**
  - Filter early and server‑side on indexed fields (e.g., `modifiedon >= :checkpoint`).
  - Project only required columns to reduce payload size.
  - Use change tracking where supported for efficient deltas.
- **Avoid**
  - Large, unfiltered scans during peak hours.
  - `expand` on large n:1 or 1:n relationships for bulk pulls.
  - Client‑side filtering after fetching broad result sets.
  - Overly frequent schedules that overlap or thrash caches.

---

## 9. Operational limits and constraints

- **Payload size limits**: Respect API request/response size caps. Split batches accordingly.
- **Timeouts**: Default server timeouts may cap long‑running queries-prefer paging over single massive requests.
- **Attachment/BLOB fields**: Large binary or long text columns dramatically reduce per‑request record counts; exclude unless needed.
- **Plug‑ins/business rules**: Write throughput is gated by synchronous plug‑ins and validation logic. Evaluate which can be asynchronous or bypassed for bulk operations.

---

## 10. Observability and capacity planning

- **Measure**
  - Request rate, average/percentile latency, success/error codes (429, 5xx), and payload sizes.
  - Per‑entity throughput and queue backlog depth.
- **Alert**
  - On consecutive throttling events, retry storms, or SLA breaches (e.g., job exceeds window by \(>20\%\)).
- **Plan**
  - Forecast capacity using historical growth and peak factors. Test at \(1.5\times\) expected peak volume.
  - Keep a staging environment with comparable data cardinality and realistic latency.

---

## 11. Example tuning playbook

1. Start with:
   - Page size 1,000 (reads), batch size 200 (writes)
   - Parallelism \(p=2\) workers per entity
   - Incremental filters on `modifiedon`
2. Run a controlled pilot on two representative entities; record:
   - Throughput, latency, error rates, and throttling signals
3. Adjust:
   - If latency dominates, increase batch size until nearing payload limits
   - If throttled, reduce \(p\) or add backoff; consider staggering entities
   - If memory spikes, reduce page size or split entities by key range
4. Re‑measure; lock in the configuration and promote to production
5. Schedule:
   - Frequency ≥ observed run time × 1.2
   - Heavy jobs off‑peak; full syncs on weekends or maintenance windows

---

## 12. FAQs

- **Can I run multiple entity syncs in parallel?**
  - Yes, but separate high‑traffic entities and monitor for throttling. Avoid starting all at the same second.
- **How often should I run incremental syncs?**
  - As often as your end‑to‑end run time permits without overlap; common cadences are every 15–60 minutes.
- **What’s the best way to recover from failures?**
  - Use durable checkpoints, idempotent upserts, and resume from the last committed high‑water mark. Do not reprocess the entire window unless checkpoints are corrupted.

---

## 13. Validation checklist

- [ ] Incremental sync enabled with durable checkpoints
- [ ] Page and batch sizes within payload limits
- [ ] Parallelism tuned; no overlapping schedules
- [ ] Backoff and retry with jitter implemented
- [ ] Monitoring for 429/5xx, latency, and backlog
- [ ] Staging load tests with representative data volumes
- [ ] Change management for schema and business rule updates

---

## 14. Key recommendations

- **Prefer incremental syncs** with server‑side filters and checkpoints.
- **Right‑size batches and concurrency** to balance latency vs. throttling.
- **Avoid unfiltered scans and heavy expansions** during peak hours.
- **Validate in staging** with realistic data and latency before production rollout.
- **Instrument and adapt**: let telemetry drive tuning decisions over time.

For help tailoring these guidelines to your Dynamics environment and workload characteristics, share target entities, data volumes, average payload sizes, and any current error/latency metrics.
