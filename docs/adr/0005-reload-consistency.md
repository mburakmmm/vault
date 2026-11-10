# ADR 0005: Snapshot Publication, Validate-Before-Swap, and Reader Consistency

## Context

Configuration changes during runtime are common in cloud-native and service environments (e.g. updating credentials, adjusting logging levels, scaling thread pool limits). However, naive in-place mutation or unvalidated reloading introduces major operational hazards:
- Partial updates (e.g., host updated to new cluster while port remains old, or updating one valid key while an invalid key breaks another subsystem).
- Race conditions (concurrent reader threads or fibers observing half-constructed dictionaries or uncoerced types).
- Silent failure or crash on corrupt inputs, causing services to enter undefined states.

## Decision

We adopt a **Validate-Before-Swap, Last-Known-Good** publication model:

1. **Candidate Build Off to the Side:**
   When `ReloadManager.reload()` is invoked, a candidate configuration snapshot is constructed completely independent of the running application snapshot. All merge precedence, type coercions, interpolation graphs, cycle detection, secret classification, and aggregate schema constraints are evaluated on the candidate.
2. **Atomic Publication on Success:**
   Only when the entire candidate configuration passes all validation checks without error, the candidate snapshot is published by updating the manager's active snapshot reference.
3. **Fail-Closed Last-Known-Good Preservation:**
   If any validation error, syntax error, cycle, or constraint violation occurs during the reload attempt:
   - A structured `ReloadResult(success=False, error_code=..., error_message=...)` is returned.
   - The active published snapshot remains **100% intact and unchanged**.
   - No partial or half-merged state is ever published to consumers.
4. **Reader Consistency & Immutability:**
   Published `ConfigSnapshot` instances are strictly immutable. Active callers holding a reference to an earlier snapshot continue reading valid, consistent state without data races or use-after-free hazards. Future reads through `manager.snapshot()` obtain the newly published snapshot.
5. **Seqlock Concurrent Reader/Writer Barrier Protocol (RELOAD-10 / RELOAD-11 / RELOAD-12 / RELOAD-13 / RELOAD-14):**
   - An `_atomic_seq: AtomicInt` sequence counter guards the publication transition.
   - Writer transaction increments sequence to odd before updating `_atomic_version` and `_published`, and increments to even when publication completes.
   - Reader verification loop (`get_published()`, `snapshot()`, `current_snapshot`) verifies `seq1 == seq2 and (seq1 & 1) == 0`.
   - **Fail-Closed Boundary (RELOAD-11):** If a concurrent write transaction is stalled or torn read persists past `max_seqlock_retries` (default: 1000 attempts), readers fail closed by raising `ConfigError("CONFIG_CONCURRENT_READ_TIMEOUT")`. Readers never receive an unverified snapshot.
   - **Hybrid Spin-Backoff & Monotonic Timeout (RELOAD-14):** Readers employ a sub-microsecond tight spin for the first 10 attempts, transitioning to 1ms sleep backoff bounded by `seqlock_timeout_ms` (default: 250ms monotonic clock).
   - **Full Transactional Rollback (RELOAD-13):** In case of any mid-publish exception or failure, `rollback_transaction()` restores `_atomic_version` and `_published` to prior known-good state before restoring `_atomic_seq` to even parity and releasing the CAS lock.
6. **NOX Concurrency & Memory Model Boundaries (CONCUR-03 / CONCUR-04):**
   - **Within Worker Process:** Async fibers and M:N work-stealing tasks safely access published snapshots via atomic hardware CAS and seqlock barriers.
   - **Across Shared-Nothing Workers:** As defined by NOX runtime architecture (QBE shared-nothing heaps), cross-worker communication utilizes primitive message channels (`ThreadChannel[T]`) or independent `ReloadManager` instances per worker (demonstrated in `examples/worker_threads.nox`).

## Consequences

- **Safety:** Eliminates inconsistent partial configurations and ensures production reliability during runtime configuration reloads.
- **Traceability:** Every published snapshot carries a deterministic monotonic version identifier (`v1`, `v2`, etc.).
- **Memory:** Candidates that fail validation are garbage-collected without side effects on process state.
