# Changelog

All notable changes to the `vault` library will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
