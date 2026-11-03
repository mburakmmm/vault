# Changelog

All notable changes to the `vault` library will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
