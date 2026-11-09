# VAULT Remediation Plan & TODO Matrix

---

## ✅ VAULT v1.0.2 Patch Release (Completed & Verified)

Addressing findings from the independent security & architectural review:

### 🚨 P0: Critical Security & Invariant Defenses

- [x] **SEC-06: Unicode Codepoint Redaction Alignment**
  - **File:** `secrets/redact.nox`, `tests/redaction_test.nox`
  - **Remediation:** Replaced byte-offset searching with pure Unicode codepoint searching (`_find_codepoint_index()`), ensuring search, slicing, and reconstruction all operate on consistent codepoint offsets. Added `test_unicode_multibyte_redaction_alignment()` verifying Turkish characters (`İstanbul`, `ş`, `ç`).
  - **Status:** Complete & Verified (PASS)

- [x] **SEC-07: Environment-Independent OS & Path Confinement**
  - **File:** `sources/files.nox`, `tests/security_test.nox`
  - **Remediation:** Eliminated environment variable reliance (`OS`, `WINDIR`). Platform path semantics are determined strictly from canonical path grammar (`canonicalize(".")`); any path starting with `/` is unconditionally treated as Unix case-sensitive.
  - **Status:** Complete & Verified (PASS)

### 🔒 P1: Snapshot Immutability, Concurrency & Network Hardening

- [x] **IMMUT-02: Complete Snapshot Encapsulation & Mutation Defense**
  - **File:** `core/snapshot.nox`
  - **Remediation:** Added defensive accessor `get_entry()` returning isolated cloned `ConfigEntry` objects and `clone()` for safe replication, preventing external callers from corrupting published snapshots.
  - **Status:** Complete & Verified (PASS)

- [x] **RELOAD-01: Concurrent Reload Coordination & Task Stress Test**
  - **File:** `reload/manager.nox`, `tests/reload_test.nox`
  - **Remediation:** Added `test_concurrent_reader_consistency_during_reload()` verifying that reader tasks reading concurrent snapshots observe monotonic versions and valid state without race conditions or partial swaps.
  - **Status:** Complete & Verified (PASS)

- [x] **NET-01: HTTPS URL Key Validation & Byte-Based Response Bounds**
  - **File:** `providers/remote_http.nox`, `tests/remote_provider_test.nox`
  - **Remediation:** Added `_validate_secret_key_path()` rejecting path traversal and query injection characters (`/`, `\`, `?`, `#`, space), and enforced the 64 KiB ceiling using `nox.strings.byte_len(resp.body)`. Added `test_key_traversal_and_injection_rejection()`.
  - **Status:** Complete & Verified (PASS)

- [x] **CONF-01: Order-Independent `with_allowed_root()` in ConfigBuilder**
  - **File:** `core/builder.nox`, `sources/interface.nox`, `sources/*.nox`, `tests/security_test.nox`
  - **Remediation:** Added `set_allowed_root()` virtual method to `ConfigSource` and implemented across all file/dotenv sources. `with_allowed_root()` retroactively propagates root confinement to earlier registered sources. Added `Test 5: Order-Independent with_allowed_root()`.
  - **Status:** Complete & Verified (PASS)

### ⚡ P2: Performance & Scalability

- [x] **PERF-01: Large-Scale Benchmark Suite**
  - **File:** `tests/benchmark_test.nox`
  - **Remediation:** Added automated performance benchmark verifying 1,000+ keys across memory, dotenv, and JSON with 3-level chained variable interpolation and schema validation completing in 7 ms (< 1500 ms SLA).
  - **Status:** Complete & Verified (PASS)

---

## ✅ VAULT v1.0.1 (Completed)
- [x] SEC-01: Linux & Unix Root Confinement Case-Sensitivity
- [x] SEC-02: Dotenv Root Confinement Conformance
- [x] SEC-03: Real `nox.http` Transport for Remote HTTPS Provider
- [x] SEC-04: URL Allowlist SSRF Parser & Userinfo (@) Rejection
- [x] IMMUT-01: Snapshot Defensive Copying & Mutation Protection
- [x] SEC-05: Strict Secret Retrieval API Separation
- [x] ENC-01: UTF-8 Codepoint vs Byte Indexing in .env Parser
- [x] ORD-01: Schema Defaults Prior to Interpolation
- [x] VAL-01: Validation of Injected Schema Default Values
- [x] ROB-01: Single-Pass Forward Scanning in Redaction
- [x] ROB-02: RFC 8259 Compliant JSON String Escaping
- [x] REL-01: Fix Release Pipeline Archive Generation
- [x] CI-01: Dual Backend Verification (LLVM & QBE on 4 platforms)
- [x] VER-01: Bump Version to v1.0.1
