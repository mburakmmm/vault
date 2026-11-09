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

## Consequences

- **Safety:** Eliminates inconsistent partial configurations and ensures production reliability during runtime configuration reloads.
- **Traceability:** Every published snapshot carries a deterministic monotonic version identifier (`v1`, `v2`, etc.).
- **Memory:** Candidates that fail validation are garbage-collected without side effects on process state.
