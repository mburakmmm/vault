# Changelog

All notable changes to the `vault` library will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.4] - 2026-11-09 (Total Snapshot Encapsulation, Hardware Atomic CAS Concurrency & SSRF Hardening)

### Fixed
- **Complete Snapshot Secret Encapsulation (IMMUT-04):**
  - Transformed `ConfigSnapshot.entries` into a `@property` returning an isolated defensive dictionary where every secret entry has its underlying value strictly redacted to `[REDACTED]`.
  - Stored raw entries in private `self._entries`, preventing external callers from bypassing secret gate mechanisms via direct dictionary access (e.g. `snapshot.entries["KEY"].val.s`).
  - Mutating the dictionary or entries returned by `snapshot.entries` has zero effect on the internal snapshot state.
  - Added dedicated test `test_snapshot_secret_entry_encapsulation` in `tests/redaction_test.nox`.
- **Hardware-Level Atomic CAS Concurrency Guard (RELOAD-03):**
  - Replaced boolean reentrancy flags in `ReloadManager` with hardware atomic `nox.atomic.new_bool(False)`.
  - Guarded critical reload transitions with atomic `compare_and_swap(False, True)` to eliminate multi-threaded race conditions during concurrent reloads.
  - Added cooperative yields (`nox.time.sleep_ms(1)`) in async reader and writer stress tests to guarantee interleaved task scheduling across both cooperative (QBE) and parallel (LLVM) backends.
- **Obfuscated IP & Pre/Post-Resolution SSRF Hardening (NET-03):**
  - Implemented `is_numeric_or_obfuscated_host()` rejecting DWORD/decimal integer hosts (e.g. `https://2130706433/`), octal octets (`https://0177.0.0.1/`), abbreviated IPs (`https://127.1/`), and hex-encoded IPs (`0x7f000001`).
  - Implemented `validate_resolved_ip()` rejecting all loopback subnets (`127.0.0.0/8`), RFC 1918 private ranges, carrier-grade NAT (`100.64.0.0/10`), IPv6 loopbacks/link-local/unique-local, and cloud link-local metadata (`169.254.169.254`).
  - Added comprehensive test coverage in `tests/remote_provider_test.nox`.

### Added
- **Statistical Multi-Run Microbenchmark (TEST-02):**
  - Added `test_statistical_benchmark_multirun` in `tests/benchmark_test.nox` performing 1 unmeasured warm-up run + 10 measured runs of 1,000 keys across schema validation and interpolation, reporting `min`, `max`, and `avg` latency telemetry.



## [1.0.3] - 2026-11-09 (Snapshot Encapsulation, True Concurrency & SSRF Hardening)

### Fixed
- **Strict Secret Gate on `get_entry()` & Safe Metadata Inspection (IMMUT-03):**
  - Updated `ConfigSnapshot.get_entry(key)` to fail closed (`ERR_CONFIG_SECRET_SOURCE_DENIED`) on keys classified as secrets, eliminating the API-boundary bypass where `.val` could disclose secrets without explicit authorization.
  - Added `ConfigSnapshot.get_secret_entry(key: str) -> ConfigEntry` for intentional and traceable secret retrieval.
  - Added `ConfigSnapshot.get_metadata(key: str) -> ConfigEntry` returning complete entry metadata while strictly masking `.val = make_string("[REDACTED]")` for secrets.
  - Added regression test `test_snapshot_secret_entry_encapsulation` in `tests/redaction_test.nox`.
- **True Concurrent Multi-Task Reload Stress Testing & Reentrancy Guard (RELOAD-02):**
  - Added an atomic `is_reloading: bool` guard in `ReloadManager.reload()` to reject reentrant or overlapping reload executions with error code `CONFIG_RELOAD_IN_PROGRESS`.
  - Added `test_true_concurrent_readers_and_writer_stress` in `tests/reload_test.nox` running 3 concurrent reader tasks across 600 continuous reads concurrently with an active writer reload using native `spawn` and `await`, proving zero torn reads or intermediate states.
- **Advanced SSRF Protection & Hostname Sanitization (NET-02):**
  - Hardened `validate_https_endpoint()` in `providers/remote_http.nox` to unconditionally reject:
    - IPv6 loopbacks (`[::1]`, `::1`, `[0:0:0:0:0:0:0:1]`)
    - Alternative IPv4 loopbacks (`127.0.0.0/8` subnet: `127.0.0.2`, `127.1.2.3`, etc.)
    - Localhost domains (`localhost`, `*.localhost`)
    - IPv6 link-local (`[fe80:*`) and Unique Local IPv6 (`[fc00:*`, `[fd00:*`)
    - IPv4-mapped IPv6 (`[::ffff:*`)
    - RFC 1918 private IPv4 networks (`10.*`, `192.168.*`, `172.16.*`–`172.31.*`)
    - URL-encoded characters (`%`) in authority to prevent hostname obfuscation.
  - Hardened `_validate_secret_key_path()` to reject URL-encoded sequences (`%`), directory traversal elements (`..`), and null bytes (`\0`).
- **Dynamic Profile Switching State Consistency (CONF-02):**
  - Fixed `ConfigBuilder.with_profile()` so that switching from `"production"` to `"development"` or `"test"` immediately resets `self.is_production = False`, ensuring builder state integrity.
  - Added `test_dynamic_profile_switching_state_consistency` in `tests/profiles_test.nox`.

### Added
- **1,000-Field Comprehensive Schema Benchmark (TEST-01):**
  - Added `test_large_scale_schema_benchmark_1000_fields` in `tests/benchmark_test.nox` testing 1,000 declared schema fields (500 strings, 300 integers with range bounds, 200 booleans) with type validation, bounds checks, and required rules. Executed in 1 ms.
- **Accurate Documentation of Test Suite Coverage (DOCS-01):**
  - Updated `README.md` to accurately document all 20 passing test suites including `tests/benchmark_test.nox`.

## [1.0.2] - 2026-11-09 (Security, Immutability & Concurrency Hardening)

### Fixed
- **Unicode Multibyte Codepoint Redaction Alignment (SEC-06):**
  - Replaced byte-offset `nox.strings.index_of()` searching in `secrets/redact.nox` with codepoint-aligned `_find_codepoint_index()`.
  - String slicing and diagnostic reconstruction now operate strictly on Unicode codepoints, preventing off-by-one/byte-boundary slice misalignments when non-ASCII multibyte characters (such as Turkish `ş`, `ğ`, `ı`, `İ`, `ö`, `ü`, `ç`) precede secret values in text.
  - Added dedicated test `test_unicode_multibyte_redaction_alignment` in `tests/redaction_test.nox`.
- **Environment-Independent OS Runtime Detection for Path Confinement (SEC-07):**
  - Eliminated spoofable environment variable lookups (`OS=Windows_NT`, `WINDIR`, `SystemRoot`) in `sources/files.nox`.
  - Added `is_windows_runtime()` using canonicalized directory inspection (`canonicalize(".")`); any path starting with `/` is unconditionally treated as POSIX case-sensitive, preventing root confinement bypasses in non-Windows environments.
- **Defensive Snapshot Immutability (IMMUT-02):**
  - Added `ConfigSnapshot.get_entry(key: str) -> ConfigEntry` returning an isolated defensive clone (`clone_config_entry()`), preventing external callers from mutating internal `ConfigEntry` objects or underlying values.
  - Added `ConfigSnapshot.clone() -> ConfigSnapshot` for safe snapshot replication.
- **URL Path Traversal & Byte-Based Response Bounds in Remote Provider (NET-01):**
  - Added strict key validation `_validate_secret_key_path(key)` in `providers/remote_http.nox`, rejecting keys with `/`, `\`, `?`, `#`, or whitespace to prevent remote endpoint path traversal and query string injection.
  - Replaced character length check with exact `nox.strings.byte_len(resp.body) > MAX_REMOTE_RESPONSE_SIZE` to enforce the 64 KiB ceiling in actual network bytes.
  - Added `test_key_traversal_and_injection_rejection` in `tests/remote_provider_test.nox`.
- **Order-Independent Root Confinement Propagation (CONF-01):**
  - Added `set_allowed_root(root: str)` virtual method to `ConfigSource` and implemented it across `DotenvSource`, `JsonSource`, `TomlSource`, and `YamlSource`.
  - Calling `ConfigBuilder.with_allowed_root()` now retroactively propagates to already registered sources and newly added sources, guaranteeing confinement regardless of call order.
  - Added `Test 5: Order-Independent with_allowed_root` in `tests/security_test.nox`.
- **Atomic Reload Snapshot Publication & Concurrent Reader Consistency (RELOAD-01):**
  - Verified atomic snapshot publishing in `ReloadManager` ensuring readers never observe intermediate or half-swapped state during active reloads.
  - Added `test_concurrent_reader_consistency_during_reload` in `tests/reload_test.nox`.

### Added
- **Large-Scale Performance Benchmark Suite (PERF-01):**
  - Added `tests/benchmark_test.nox` loading 1,000 keys across memory, dotenv, and JSON sources with 3-level chained variable interpolation and schema validation, executing in under 10 ms (well within the 1500 ms SLA).

## [1.0.1] - 2026-11-09 (Security, Correctness & Reliability Patch)

### Fixed
- **Root Confinement Case Sensitivity (SEC-01):** Made path component comparison in `sources/files.nox` platform-aware. Linux, macOS, and Unix now enforce strict case-sensitivity (preventing `/data/config` vs `/data/Config` bypass), while Windows retains case-insensitivity.
- **Dotenv Root Confinement Conformance (SEC-02):** `DotenvSource` in `env/source.nox` now enforces `allowed_root` confinement through `read_bounded_file()` and is plumbed through `ConfigBuilder.with_allowed_root()`.
- **Remote HTTPS Provider Transport & SSRF Hardening (SEC-03, SEC-04):**
  - Added real `StandardHttpTransport` in `providers/remote_http.nox` using `nox.http.request` with redirect inspection and timeout bounds.
  - Implemented strict URL authority parsing that unconditionally rejects userinfo (`@`) credentials in URLs, stopping userinfo spoofing bypasses.
- **Snapshot Immutability via Defensive Copying (IMMUT-01):** `ConfigSnapshot` now performs deep defensive cloning of `ConfigEntry` objects and dictionaries upon publication, protecting runtime snapshots against external mutation.
- **Strict Secret Retrieval API Separation (SEC-05):** Calling `get_string()`, `get_int()`, `get_float()`, `get_bool()`, or `get_list()` on a key classified as a secret now fails closed (`ERR_CONFIG_SECRET_SOURCE_DENIED`). Secrets must be intentionally retrieved via `get_secret()` or `get_secret_value().reveal()`.
- **Unicode UTF-8 Codepoint Parsing in .env (ENC-01):** Refactored quoted and unquoted value parsing in `env/parser.nox` to use codepoint indexing and slicing rather than mixing byte offsets with character counts. Added regression tests verifying Turkish characters (`İstanbul`, `Türkçe`, `şemsiye`, `öğrenci`, `çözüm`).
- **Schema Defaults Ordering & Constraint Validation (ORD-01, VAL-01):**
  - Injected schema default values prior to `resolve_interpolations()`, enabling expressions like `${DEFAULT_HOST}` to resolve correctly against schema defaults.
  - Added full type coercion and bounds/range constraint validation for injected default values.
- **Single-Pass Redaction & JSON String Escaping (ROB-01, ROB-02):**
  - Replaced recursive `index_of()` string substitution with single-pass forward scanning in `secrets/redact.nox`, preventing infinite loops if a sentinel matches part of `[REDACTED]`.
  - Replaced handwritten JSON string escaping with standard `nox.json.dump_string()` to ensure complete RFC 8259 escaping of all ASCII control characters (`\u0000`–`\u001f`).
- **Release Pipeline Archive Packaging (REL-01):** Fixed `release.yml` archive generation using `git archive` to `$RUNNER_TEMP`, eliminating the `tar: .: file changed as we read it` runner error.
- **Dual-Backend CI Verification (CI-01):** Updated GitHub Actions workflow to explicitly execute all 19 test suites against both `noxc test --backend qbe` and `noxc test --backend llvm` on all supported runner platforms (macOS ARM64, Linux x64, Linux ARM64, Windows x64).

## [1.0.0] - 2026-11-09 (Production Stable Release)

### Added
- Comprehensive regression scenario matrix test suite (`tests/regression_matrix_test.nox`) verifying full compliance with AGENTS.md §19 acceptance criteria:
  - `ENV` (01-08): Pure parsing, CRLF/BOM normalization, unquoted, double-quoted, single-quoted, escape sequences, duplicate key rejection, and bounded resource limits.
  - `MERGE` (01-05): Strict 9-tier deterministic source precedence, origin provenance tracking, and missing/null/empty distinctions.
  - `SCHEMA` (01-05): Typed schema validation (`int`, `float`, `string`, `bool`, `list`), numeric range bounds, length constraints, aggregate multi-error collection without early aborts, and strict coercion.
  - `INTERP` (01-04): Graph-based variable resolution, cycle detection, multi-hop chains, depth/size limits, and automatic transitive secret taint propagation.
  - `SECRET` (01-04): Complete leak prevention in logs, `str()`, JSON exports, exceptions, and safe diagnostics, with authorized `get_secret()` retrieval and defense-in-depth string redaction.
  - `FILE` (01-05): Root confinement, directory traversal rejection (`../` and symlink escapes), and 1 MiB file size limits.
  - `FORMAT` (01-02): Type-preserving JSON, TOML 1.0, and YAML bridges with structured container preservation.
  - `PROFILE` (01-02): Profile layering (`development`, `test`, `staging`, `production`), filename mapping, and production strictness.
  - `PROVIDER` (01-04): Local providers (memory, env, file) and remote HTTPS provider with mock transport, SSRF defense, redirect protection, bounded retries, and token non-disclosure.
  - `RELOAD` (01-02): Validate-before-swap atomic reloads, monotonic version progression, and last-known-good state preservation on candidate errors.
  - `CLI` (01-05): Standalone CLI tool (`cli/main.nox`) supporting `validate`, `explain`, `check-env`, `version`, and side-effect-free `--help`.
  - `CI` (01-03): Deterministic automated test execution across 19 test suites verified 100% green on **both** LLVM (`zig 0.17.0`) and QBE backends.
- Sealed v1.0.0 API contracts, documentation, and architecture decision records.

## [0.9.8] - 2026-11-09

### Added
- Independent ecosystem adapters in `adapters/` for NOX ecosystem projects:
  - `adapters/aether.nox`: HTTP server options (`host`, `port`, timeouts, payload size limits) with `AetherConfig` and `build_aether_schema`.
  - `adapters/nyx.nox`: Web application options (`app_name`, `base_path`, `session_secret`, `debug_mode`) with `NyxConfig`, `build_nyx_schema`, and secret disclosure protection in `to_string()`.
  - `adapters/feather.nox`: Database connection settings (`driver`, `host`, `port`, `database`, `user`, `password`, `pool_size`, `ssl_mode`) with `FeatherDbConfig`, `build_feather_schema`, and masked `connection_string()`.
  - `adapters/noxen.nox`: Worker service options (`worker_name`, `concurrency`, `queue_name`, `max_retries`) with `NoxenConfig` and `build_noxen_schema`.
  - `adapters/nova.nox`: Async event broker settings (`broker_url`, `client_id`, `api_token`, `retry_interval_ms`) with `NovaConfig`, `build_nova_schema`, and token redaction.
- Public core isolation: VAULT core does not transitively import adapters or require third-party libraries.
- Test suite `tests/adapters_test.nox` covering schema declaration, snapshot extraction, redaction guarantees, and core package isolation.
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends (18/18 test suites passing).

## [0.9.5] - 2026-11-09

### Added
- Optional CLI tool and executable entry point (`cli/main.nox`) supporting `validate`, `explain`, `check-env`, `version`, and `help`.
- `vault validate` command (**`CLI-01`**) validating configuration against schema and profile, returning status code `0` for success and `1` for failures.
- Zero side-effect `vault --help` (**`CLI-02`**) ensuring help invocations never open network connections, read secret stores, or evaluate config files.
- `vault explain <key>` command with safe provenance inspection without secret disclosure.
- `vault check-env` command verifying presence of required environment variables without disclosure.
- Test suite `tests/cli_test.nox` covering help safety, version inspection, valid/invalid config validation, explain output, check-env verification, and unknown command handling.
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends (17/17 test suites passing).

## [0.9.0] - 2026-11-09

### Added
- Remote HTTPS secret provider (`providers/remote_http.nox`) with pluggable `HttpTransport` abstraction.
- Hermetic test fake `MockHttpTransport` ensuring zero live network calls in standard CI (**`PROVIDER-01`**).
- SSRF and protocol protection (`validate_https_endpoint`) blocking non-HTTPS schemes (`http://`, `ftp://`), cloud metadata IP addresses (`169.254.x.x`), unroutable IPs (`0.0.0.0`, `255.255.255.255`), and hosts outside the explicit allowlist.
- Pre-request redirect validation (**`PROVIDER-02`**) preventing credential leakage by blocking HTTP 301/302 redirects to unapproved or non-HTTPS destinations before sending authentication tokens.
- Bounded retries and timeout handling (**`PROVIDER-03`**) with exponential fail-closed behavior returning `CONFIG_PROVIDER_UNAVAILABLE` on exhausted retries without hanging.
- Safe auth failure reporting (`CONFIG_SECRET_SOURCE_DENIED`) on 401/403 responses guaranteeing authentication tokens are never exposed in exception messages, logs, or diagnostics.
- Response payload ceiling enforcement (64 KiB) returning `CONFIG_SIZE_LIMIT` on oversized payloads.
- Integration with `ConfigBuilder` via `with_secret_provider` mapping fetched remote secrets into rank 7 with automatic redaction and secret tagging.
- Test suite `tests/remote_provider_test.nox` covering SSRF rejection, hermetic mock fetching, redirect protection, bounded retries, auth disclosure protection, response size limits, and builder integration.
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends (16/16 test suites passing).

## [0.8.0] - 2026-11-09

### Added
- Validated reload manager (`reload/manager.nox`) with atomic snapshot swapping and validate-before-swap semantics.
- Structured reload outcome model (`ReloadResult`) reporting success flag, version number, safe error codes, and effective snapshot.
- Fail-closed rollback mechanism (**`RELOAD-01`**) ensuring that invalid candidate configurations (schema violations, cycles, missing keys) preserve the previous last-known-good snapshot wholly intact without partial state.
- Monotonic version progression (**`RELOAD-02`**) on valid reloads (`v1` -> `v2` -> `...`).
- Immutable reader consistency ensuring in-flight readers holding earlier snapshots observe consistent state without races or use-after-free hazards.
- Cloned entry loading in `MemorySource` and `merge_sources` protecting source templates against in-place mutation across reload attempts.
- Architecture Decision Record `docs/adr/0005-reload-consistency.md`.
- Test suite `tests/reload_test.nox` covering successful reload, type mismatch rollback, interpolation cycle rollback, consecutive reload metrics, and reader snapshot immutability.
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends (15/15 test suites passing).

## [0.7.0] - 2026-11-03

### Added
- Pluggable secret provider interface and result contract (`providers/interface.nox`) with `SecretProvider`, `SecretFetchResult`, and `ProviderHealth`.
- In-memory mock secret provider (`providers/memory.nox`) for deterministic testing and health reporting.
- Allowlist-only environment secret provider (`providers/env.nox`) strictly restricting OS environment queries to authorized keys.
- Root-confined file secret provider (`providers/file.nox`) with path confinement, directory traversal rejection (`FILE-01`), and 64 KiB size ceilings.
- Configuration source bridge `SecretProviderSource` (`providers/source.nox`) binding providers at precedence rank 7 with automatic `is_secret = true` tagging.
- `ConfigBuilder` extension `with_secret_provider(provider, keys, fail_closed)`.
- Test fixtures in `fixtures/secrets_dummy/` (`db_pass.txt`, `api_key.txt`).
- Test suite `tests/secret_provider_test.nox` covering memory, env, file providers, precedence rank 7, fail-closed handling, and zero-network isolation (**`PROVIDER-01`**).
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends.

## [0.6.0] - 2026-11-03 (Practical Package Milestone)

### Added
- Explicit secret model and wrapper abstraction (`secrets/mark.nox`) with `SecretValue` defaulting textual representation to `[REDACTED]`.
- Intentional and authorized secret retrieval (`secrets/reveal.nox`) via `snapshot.get_secret(key)` and `snapshot.get_secret_value(key).reveal()` (**`SECRET-03`**).
- Structured redaction and deterministic JSON serialization (`secrets/redact.nox`) with `to_redacted_json` and `snapshot.redacted_json()` (**`SECRET-02`**).
- Defense-in-depth text redaction `redact_string(text, sentinels)` (**`SECRET-01`**).
- In-memory secret cache with TTL support (`secrets/cache.nox`) supporting `set`, `get`, `evict`, `clear`, and auto-eviction of expired entries.
- Transitive secret taint redaction ensuring derived settings interpolated from secrets are cleanly redacted in JSON.
- Test suite `tests/redaction_test.nox` covering sentinel scan verification (`SECRET-01`, `SECRET-02`, `SECRET-03`, `CI-03`), TTL caching, and taint redaction.
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends.

## [0.5.0] - 2026-11-03

### Added
- Multi-environment profile management (`profiles/overlay.nox`) supporting `development`, `test`, `staging`, `production`, and sanitized custom profiles.
- Profile-aware file and `.env` resolution (`vault.<profile>.toml`, `.env.<profile>`).
- Strict production profile checks (`PROFILE-01`) enforcing rejection of hardcoded/default secrets.
- Unknown profile validation (`PROFILE-02`) preventing silent fallback to development.
- Graph-based deterministic string interpolation engine (`interpolate/resolve.nox`) with `${KEY}` resolution.
- Multi-hop reference chain resolution (`INTERP-01`, e.g. `A -> B -> C`).
- Direct (`A -> A`) and indirect (`A -> B -> A`) cycle detection (`INTERP-02`) reporting safe `CONFIG_INTERPOLATION_CYCLE` error codes without leaking values.
- Resource bounds protection (`INTERP-03`) with maximum 16 recursion hops and 128 KiB expansion limits.
- Transitive secret taint propagation (`INTERP-04`) ensuring derived settings inherit `is_secret = true`.
- Literal dollar escaping via `$$` and `${KEY:-default}` literal fallbacks.
- Integrated profile loading and interpolation in `ConfigBuilder`.
- Test suites:
  - `tests/interpolation_test.nox` (INTERP-01, INTERP-02, INTERP-03, INTERP-04, escaping, defaults, missing refs).
  - `tests/profiles_test.nox` (PROFILE-01, PROFILE-02, filename resolution, overlay precedence).
- Verified 100% green across both LLVM (`zig 0.17.0`) and QBE compiler backends.

## [0.4.0] - 2026-11-03

### Added
- Root-constrained safe file handling (`sources/files.nox`) with path-segment-aware canonicalization, directory traversal rejection, and 1 MiB size ceilings.
- JSON configuration source (`sources/json.nox`) mapping `nox.json` into type-preserving `ConfigValue` hierarchy.
- TOML configuration source (`sources/toml.nox`) mapping `nox.toml` 1.0 tables, arrays, and scalars.
- YAML configuration source (`sources/yaml.nox`) mapping `nox.yaml` mappings and sequences.
- `ConfigBuilder` extension methods `with_allowed_root`, `with_json_file`, `with_toml_file`, `with_yaml_file`, and `with_file`.
- Configuration fixtures in `fixtures/config/` (`app.json`, `app.toml`, `app.yaml`, `invalid.json`).
- Test suites:
  - `tests/json_source_test.nox` (FORMAT-01, FORMAT-02).
  - `tests/toml_source_test.nox` (FORMAT-01).
  - `tests/yaml_source_test.nox` (FORMAT-01).
  - `tests/security_test.nox` (FILE-01, FILE-03, FILE-04).

## [0.3.0] - 2026-11-03 (MVP Milestone)

### Added
- Typed schema definitions (`schema/fields.nox`) with `ConfigSchema`, `Field`, and constraints (`required`, `nullable`, `non_blank`, `min`, `max`, `secret`).
- Strict coercion engine (`schema/coerce.nox`) for integer, float, boolean (case-insensitive variants: true/false, 1/0, yes/no, on/off), and string list conversions.
- Aggregated safe validation engine (`schema/validate.nox`) returning non-sensitive deterministic `ValidationReport` and `SafeConfigDiagnostic` without leaking values.
- Fluent `ConfigBuilder` (`core/builder.nox`) supporting multi-source wiring, defaults, `.env`, allowlisted process environment, overrides, and schema validation.
- Typed snapshot getters (`get_int`, `get_float`, `get_bool`, `get_list`) in `core/snapshot.nox`.
- Test suite `tests/schema_test.nox` covering regression scenarios `SCHEMA-01` through `SCHEMA-05`.
- Full end-to-end integration test `tests/integration_test.nox` verifying the MVP milestone.

## [0.2.5] - 2026-11-03

### Added
- Core value models and presence states (`ConfigValue`, `SourceOrigin`, `ConfigEntry`, `PRESENCE_MISSING`, `PRESENCE_NULL`, `PRESENCE_EMPTY`, `PRESENCE_SUPPLIED`).
- Standard safe configuration error kinds (`core/errors.nox`).
- Source abstractions (`sources/interface.nox`, `sources/memory.nox`).
- Bounded file dotenv loader (`env/source.nox`) with 1 MiB size verification.
- Allowlist-only process environment source (`env/process.nox`) using explicit keys and `nox.os.getenv`.
- Deterministic multi-source precedence and recursive object merging (`core/merge.nox`).
- Immutable published configuration snapshot (`core/snapshot.nox`) with safe non-secret `explain(key)`.
- Architectural Decision Record `docs/adr/0002-source-precedence.md`.
- Comprehensive test suites `tests/env_process_test.nox` and `tests/merge_test.nox` covering regression scenarios `MERGE-01` to `MERGE-04` and `ENV-01` to `ENV-02`.

## [0.2.0] - 2026-11-03

### Added
- Pure `.env` parser (`env/parser.nox`) supporting `KEY=VALUE`, spaces around `=`, `export` prefix ignoring, single and double quoted values, and inline comments.
- Escape sequence support for `\n`, `\r`, `\t`, `\\`, and `\"` in double-quoted strings.
- Bounded input limits (1 MiB file size, 2,000 keys, 64 KiB value, 128 bytes key length).
- Safe, non-sensitive parse errors (`EnvParseError`) with line/column coordinates and error codes (`CONFIG_PARSE_ERROR`, `CONFIG_DUPLICATE_KEY`, `CONFIG_SIZE_LIMIT`).
- UTF-8 BOM codepoint stripping and CRLF normalization.
- Test fixtures (`fixtures/dotenv/valid.env`, `quotes.env`, `malformed.env`).
- Comprehensive parser test suite (`tests/env_parser_test.nox`) covering regression scenarios `ENV-03` to `ENV-08`.

## [0.1.0] - 2026-11-03

### Added
- Project skeleton (`nox.json`, `.gitignore`, `.env.example`, `CHANGELOG.md`).
- Core capability probes in `tests/probes/` verifying NOX v2.0.123 standard library behaviors.
- Compatibility documentation in `docs/compatibility.md`.
- Architectural Decision Record `docs/adr/0001-value-model.md`.
- Initial package smoke test `tests/package_test.nox`.
- Initial `vault.nox` public facade.
