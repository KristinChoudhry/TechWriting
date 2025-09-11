# Dynamics Connector: Scale and Performance Implications

This article outlines the performance characteristics and operational limits for the Dynamics Connector.

## Performance limits

- Throughput benchmark: approximately \(x\) records/second under typical load.
- Scheduling guidance: do not schedule syncs more frequently than one full run time.
- Anti‑patterns: avoid large, unfiltered queries; prefer incremental sync with checkpoints.
- Batch sizing: use moderate batch sizes and parallelism within documented limits.

Thoughtful scheduling, incremental strategies, and right‑sized batches reduce contention and timeouts. Validate with representative data volumes in a staging environment.
