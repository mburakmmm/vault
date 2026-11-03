# AGENTS.md — VAULT: Configuration, Environment & Secret Management for NOX

> **Status:** GREENFIELD IMPLEMENTATION BLUEPRINT — not an assertion that any listed module or API already exists.  
> **Project:** `vault` — a reusable, framework-independent NOX 2.x library.  
> **Product analogy:** `python-dotenv` + a deliberately scoped subset of Dynaconf, with typed schema validation, controlled composition and pluggable secret sources.  
> **Baseline:** use NOX v2.0.123 as the initial compiler compatibility target, then pin the exact validated compiler/runtime version in CI. Verify the current toolchain before implementation.  
> **Priority:** correctness, deterministic precedence, fail-closed configuration and secrecy before convenience.  
> **Audience:** autonomous AI coding agents, code reviewers, maintainers and CI automation.

---

## 0. Agent operating contract — mandatory

You are an implementation agent developing **VAULT** for the NOX ecosystem. Read **this entire document**, the current repository, `README.md`, `nox.json`, `nox.lock` (if present), and the relevant NOX standard-library documentation **before changing source**. Do not treat an example, proposed interface, roadmap item or changelog as proof of an implemented capability.

### Non-negotiable rules

1. **NOX first.** Write VAULT itself in `.nox` modules. Use the existing NOX standard library (`nox.os`, `nox.fs`, `nox.path`, `nox.strings`, `nox.json`, `nox.toml`, `nox.yaml`, `nox.crypto`, `nox.log`, `nox.time`, where appropriate). Do not silently require Python, Node, Go, Rust, C or Zig at runtime.
2. **Standalone package.** VAULT core must not depend on Nyx, Aether, Aura, Noxen, Nova, Rune, Feather, Atlas, Quartz, or other ecosystem projects. Consumers may use VAULT through optional adapters; development-only test infrastructure may use Quartz if available.
3. **No compiler modifications.** Work around compiler limitations in library code when safe. If something fundamentally requires a compiler/runtime change, create a precise upstream issue/probe and mark the feature BLOCKED; do not secretly modify `nox-lang`.
4. **Never invent NOX APIs.** Before implementing an operation, verify its signature against the checked-out pinned NOX version. Proposed APIs below are contracts, not executable code until a compiling probe verifies them.
5. **Safe by default.** Loading a `.env` file must **not** mutate process environment variables by default. It returns a config source/snapshot. Explicit export to `nox.os.set_var` is a separate dangerous opt-in operation and is excluded from the initial MVP.
6. **No secret leakage.** Never log, print, serialize, include in `repr`/diagnostics, include in URLs, store in crash artifacts, or commit secret *values*. This includes exception messages, stack traces, snapshots and test fixtures. Errors may contain non-sensitive key names and error codes only.
7. **Strict, deterministic precedence.** A config key has exactly one effective winning source, a traceable **non-secret** provenance record, and a defined behavior for missing, empty, null, invalid and duplicate entries.
8. **Fail closed for required settings.** In production/strict mode, a required secret or invalid typed value must cause a configuration failure, not silent fallback to a weaker source.
9. **No unsafe evaluation.** `.env` files are data, never shell scripts. Absolutely no `eval`, command substitution, executable expressions, implicit subprocesses, or `source .env`.
10. **No automatic network access.** Remote secret providers are opt-in, explicitly configured, HTTPS-only and allowlisted. Never try to discover local credentials or cloud instance metadata on startup.
11. **Limit all inputs.** Bound config size, number of keys, interpolation depth, interpolation expansion length, source count, provider responses and retries. Reject or clearly report breaches; do not hang or silently truncate values.
12. **Stable behavior on every OS.** Test Windows x64, Linux x64/ARM64 and macOS ARM64 where CI allows. Paths, CRLF, BOM and environment-key behavior must be explicitly specified.
13. **Immutable published snapshots.** Build, merge, resolve and validate off to the side. Only expose a new snapshot after *all* checks pass; on failure retain the last known good snapshot.
14. **No fake CI success.** An assertion must be executed to count as passed. A type-check-only fallback cannot substitute for a runtime test. Do not equate passing test files with individual assertions.
15. **Reproducibility.** Pin dependencies in `nox.lock`; pin compiler version, document backends and use deterministic fixtures, seeds, clocks and provider fakes for tests.
16. **Small auditable changes.** Never rewrite unrelated modules for convenience. Use phase acceptance gates. Update tests, docs and `CHANGELOG.md` with every functional change.
17. **Keep public core independent of optional storage/provider protocols.** A failure to load an optional provider must not break `import vault` when the user has not opted into it.
18. **No provider branding confusion.** The VAULT package is a NOX configuration library; it is **not** HashiCorp Vault, an encryption product, a credential broker, or an operating-system secure storage implementation. HashiCorp Vault integration, if added, is explicitly an optional adapter.

### Standard agent execution loop

For **every feature or phase**:

1. Inventory current code and tests; state which pieces already work and which do not.
2. Read current NOX API docs and write/execute a minimal capability probe for any unfamiliar language or stdlib feature.
3. Define public behavior and adversarial cases before implementing the feature.
4. Implement the smallest integrated vertical slice, not a parallel mock implementation presented as production.
5. Add positive, negative, boundary, hostile-input and cross-platform tests.
6. Run `noxc check vault.nox`, `noxc test`, and the relevant backend-specific runs; save exact commands and outcomes. If unavailable, write **NOT RUN**, not PASS.
7. Verify security invariants, including redaction, no unintended writes, no unintended network, bounded resource usage and no secret disclosure.
8. Update `README.md`, `docs/api.md`, applicable recipes, `CHANGELOG.md` and roadmap status.
9. End each change summary with **Implemented / Tested / Not tested / Known risks / Next step**.
10. Do not declare a phase finished if any mandatory acceptance criterion remains unmet.

---

## 1. Product definition and success criteria

**VAULT** is the shared configuration foundation for NOX libraries, command-line tools and backend services. It provides a reliable answer to: *Where does this setting come from? Is it valid? Can it be loaded without leaking secrets? Can it be updated safely?*

### Product pillars

| Pillar | Responsibility | Initial priority |
| --- | --- | --- |
| `vault.env` | Strict `.env` parser and explicit NOX process-environment access | P0 |
| `vault.core` | Immutable config tree, value presence, provenance and deterministic merges | P0 |
| `vault.schema` | Typed fields, required/default/nullable rules and aggregate diagnostics | P0 |
| `vault.sources` | Defaults, memory, `.env`, environment, JSON, TOML and YAML sources | P0/P1 |
| `vault.interpolate` | Controlled `${KEY}` references with cycles/depth detection | P1 |
| `vault.secrets` | Secret-marked values, redaction and non-sensitive diagnostics | P0 |
| `vault.profiles` | Explicit dev/test/staging/prod configuration overlays | P1 |
| `vault.providers` | Optional secret provider contract, caching and credential isolation | P2 |
| `vault.reload` | Validated reload with last-known-good rollback | P2 |
| `vault.cli` | Optional explicit commands: `validate`, `explain`, `check-env` | P2 |
| `vault.adapters` | Optional integrations for Aether, Nyx, Noxen, Nova, Feather and other NOX ecosystem libraries | P3 |

### Required developer experience

A NOX application can:

1. Declare a config schema (e.g., `APP_ENV`, `HTTP_PORT`, `DATABASE_URL`, `JWT_SECRET`).
2. Load `.env`, optional `.env.<profile>`, configuration documents and real process environment in a predictable order.
3. Validate all required settings, types and constraints before opening network sockets or DB connections.
4. Read typed values and receive safe missing/invalid-value errors.
5. Mark secrets and expose only redacted inspection/export views.
6. Explain **where** a key came from without exposing its value.
7. Override values deliberately in a test without polluting global OS environment.
8. Eventually opt into remote secret providers and a validated reload path.

### Explicit non-goals

- A replacement for `nox.os`, `nox.json`, `nox.toml`, `nox.yaml` or the NOX package manager.
- A command shell, executable expression engine or configuration templating language with side effects.
- A general-purpose encrypted database, password manager or KMS implementation.
- Automatic scanning/importing of every environment variable (the documented NOX API only provides `getenv(name)`, not environment enumeration).
- Full Python Dynaconf syntax/feature parity in v1.0.
- Automatic modification of `os.environ`/child-process environment during normal loading.
- Remote secret management, hot reload, file watching or cross-process coordination in the initial MVP.
- Use of secret values as metric labels, cache keys, tracing tags or file names.

### v1.0 success definition

A release is ready only when users can load and validate configuration on supported platforms, achieve deterministic precedence, consume typed values, inspect safe provenance, use optional approved secret adapters without risking disclosure, and pass malicious-input and production-mode failure tests. v1.0 requires green CI and a documented compatibility matrix, not just a feature checklist.

---

## 2. Platform facts — verify against the pinned NOX release

Consult and keep aligned with:

- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/os.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/fs.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/path.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/json.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/toml.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/yaml.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/log.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/crypto.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/stdlib/http.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/tools/testing.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/tools/packages.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/language/concurrency.md
- https://github.com/mburakmmm/nox-lang/blob/main/docs/reference/security.md

### Verified design constraints

- `nox.os.getenv(name)` returns `str` or raises `OsError` when absent. It has **no default argument**; `nox.os.set_var(name, value)` alters the current process and its child processes. VAULT must use `try`/`except OsError` and must not auto-export.
- `nox.fs.read_to_string(path)` can read files but buffers the full file; inspect `nox.fs.metadata(path).size` first and reject oversized files before reading. `nox.fs` performs no sandbox validation.
- `nox.path.canonicalize(path)` resolves symlinks and requires an existing path. Restrict untrusted paths to an explicitly allowed canonical root with a **path-segment-aware** comparison; a string-prefix check alone (`/app/config` vs `/app/config-evil`) is insufficient.
- `nox.json.parse` produces `JsonValue`; the number representation can lose integer precision for large JSON integers. Use strict ranges and document numeric boundaries.
- `nox.toml.parse` supports TOML 1.0 and `TomlValue` nodes, including native integer types and tables.
- `nox.yaml.parse` supports a practical YAML subset, not necessarily every YAML feature. Reject unsupported input rather than attempting silent compatibility.
- `nox.log.redact` and `nox.log.mask_assignments` exist, but VAULT must build **structured** redaction first; no regex-only approach can guarantee full secrecy for arbitrary secrets.
- `nox.crypto` supplies hashing, HMAC, CSPRNG and constant-time equality; **it does not supply an encryption cipher**. Do not invent `encrypt`/`decrypt` or claim secrets are encrypted at rest.
- `nox.http` requires the network capability; remote adapters must not be imported/initialized in a pure configuration-only application unless selected.
- `noxc test` discovers `*_test.nox`; test files need real executable assertions. `noxc check` checks type correctness but **does not execute code**.
- Compiler backends can differ in scheduling; do not assume concurrency semantics from a single backend.
- Hosted OS, filesystem and networking operations are unavailable or capability-restricted in freestanding/sandboxed profiles. The pure parser and validator should still be useful without OS calls when possible.

### Mandatory Phase 0 capability probes

Before deciding the exact API, compile and run probes for:

1. Custom exception classes; preserving message-free failure codes without sensitive data.
2. `dict[str, T]`, `list[T]`, nested typed classes and safe value copying.
3. `str | None` and explicit distinction among missing, null and empty.
4. Function-typed callbacks or structural protocols for source/provider injection.
5. `nox.os.getenv` absence behavior and `set_var` child inheritance in a test process only.
6. `nox.fs.metadata`, oversized-file rejection, path canonicalization and symlink behavior.
7. JSON/TOML/YAML parse error behavior and nested-value traversal.
8. String slicing, Unicode/byte-length handling, Windows CRLF and UTF-8 BOM.
9. Exception behavior and formatting under both QBE and LLVM.
10. Any proposed atomic snapshot swapping or locking primitive, before designing concurrent reload.

Record actual probe paths/results in `docs/compatibility.md`; adapt this blueprint if the installed compiler differs.

---

## 3. Architecture and dependency boundaries

```text
                         +---------------------------+
                         | Application / service     |
                         | Aether / Nyx / CLI / etc. |
                         +-------------+-------------+
                                       |
                         +-------------v-------------+
                         | vault.nox public facade    |
                         +-------------+-------------+
                                       |
        +------------------------------+-------------------------------+
        |                              |                               |
  +-----v------+                +------v------+                 +------v------+
  | vault.core |<---------------| vault.schema|                 | vault.secrets|
  | values     |                | validate    |                 | labels, safe |
  | snapshots  |                | typed views |                 | diagnostics  |
  +-----+------+                +-------------+                 +-------------+
        |
  +-----v---------------------------------------------------------------+
  | vault.sources / vault.env / vault.interpolate / vault.profiles      |
  | memory, dotenv, OS environment, JSON, TOML, YAML, explicit overlays |
  +--------------------------+------------------------------------------+
                             |
                    +--------v---------+
                    | optional providers|  env, file, HTTPS/Vault/KMS adapters
                    +------------------+

  Optional: vault.reload, vault.cli, vault.adapters.*
```

### Dependency direction

- `core` has no I/O or network dependencies and defines immutable snapshots, source interfaces and diagnostics.
- `env.parser` parses **strings only**; `env.source` performs bounded filesystem access; `env.process` reads explicitly requested environment keys.
- `schema` depends on `core`; it must not query OS environment or open files during validation.
- `interpolate` consumes already-loaded values. It has no I/O, command execution or hidden provider access.
- `sources.json/toml/yaml` parse document text through NOX stdlib into a common type-preserving intermediate tree.
- `secrets` defines metadata, safe value access and redaction; provider adapters may depend on it, not vice versa.
- `providers` are optional and depend on core contracts; they must not become transitive mandatory imports for basic `.env` use.
- `reload` builds a complete candidate snapshot and swaps only on success; callbacks observe either old or new snapshot, never half-merged state.
- `cli` consumes public library APIs, not internal private parsers.
- Adapters must not induce core → framework dependencies. Avoid circular imports.

### Architecture decision records (ADRs)

Create `docs/adr/` and document at least:

- `0001-value-model.md`: nested tree vs flattened keys, missing/null/empty semantics.
- `0002-source-precedence.md`: merge order and secret-key policy.
- `0003-dotenv-grammar.md`: accepted syntax and deliberate exclusions.
- `0004-secret-handling.md`: redaction, memory limitations and remote-provider trust.
- `0005-reload-consistency.md`: snapshot publication and concurrent readers.
- `0006-provider-auth.md`: credential acquisition, endpoints and rotation.
- `0007-platform-capabilities.md`: hosted vs sandbox/freestanding support.

---

## 4. Repository layout — implementation target

```text
vault/
  AGENTS.md
  README.md
  CHANGELOG.md
  LICENSE
  nox.json
  nox.lock
  .gitignore
  .env.example                      # dummy values only; no credentials
  vault.nox                         # stable public facade
  core/
    types.nox                       # ConfigValue, presence, origin, metadata
    errors.nox                      # safe error kinds/codes
    snapshot.nox                    # immutable validated config snapshot
    merge.nox                       # deterministic per-key source precedence
    options.nox                     # limits, strictness, profile
  env/
    parser.nox                      # pure dotenv parser
    source.nox                      # bounded .env file source
    process.nox                     # get explicitly named OS env keys
  sources/
    interface.nox                   # source contract / result record
    memory.nox                      # test overrides + defaults
    json.nox                        # nox.json bridge
    toml.nox                        # nox.toml bridge
    yaml.nox                        # nox.yaml bridge
    files.nox                       # safe rooted-file lookup
  schema/
    fields.nox                      # Field, types, constraints, defaults
    validate.nox                    # full aggregate validation
    coerce.nox                      # strict parsing / conversion
  interpolate/
    resolve.nox                     # reference graph, cycle detection
  secrets/
    mark.nox                        # secret classification / provenance
    reveal.nox                      # intentional access contract
    redact.nox                      # safe diagnostic / export / logging
    cache.nox                       # opt-in secret cache with TTL
  profiles/
    overlay.nox                     # base/profile layering
  providers/
    interface.nox                   # provider contract; optional
    env.nox                         # named environment secret keys
    file.nox                        # bounded, root-restricted secret files
    remote_http.nox                 # optional, approved HTTPS-only adapter
    hashicorp.nox                   # optional future Vault integration
    cloud.nox                       # optional future cloud provider adapters
  reload/
    manager.nox                     # validate-before-swap
  cli/
    main.nox                        # optional 'vault' executable
  adapters/
    aether.nox
    nyx.nox
    feather.nox
    noxen.nox
    nova.nox
  examples/
    basic_env.nox
    typed_config.nox
    multiple_sources.nox
    profiles.nox
    secrets.nox
    service_bootstrap.nox
  tests/
    env_parser_test.nox
    env_process_test.nox
    merge_test.nox
    schema_test.nox
    interpolation_test.nox
    json_source_test.nox
    toml_source_test.nox
    yaml_source_test.nox
    profiles_test.nox
    redaction_test.nox
    secret_provider_test.nox
    reload_test.nox
    security_test.nox
    integration_test.nox
  fixtures/
    dotenv/
    config/
    secrets_dummy/
  docs/
    api.md
    recipes.md
    security.md
    compatibility.md
    migration.md
    adr/
  .github/workflows/ci.yml
```

Only introduce directories/files when their implementation phase begins. Don't commit empty stubs that create the appearance of progress. Do not add an external dependency merely to implement a parser already provided by NOX stdlib.

---

## 5. Core data model and invariants

### Value types

The library must distinguish:

- **MISSING**: no source supplied a value.
- **NULL**: an explicitly null value from JSON/TOML/YAML (where the source format permits); not the same as missing.
- **EMPTY_STRING**: `""`, explicitly supplied.
- **STRING**, **INTEGER**, **FLOAT**, **BOOLEAN**, **ARRAY**, **OBJECT**: typed values from structured sources.
- **SECRET**: *metadata/classification on any supported value*, not an invented scalar kind that discards the underlying type.

Start with a type-preserving `ConfigValue` tree or an equivalent verified NOX-friendly representation. **Do not normalize every JSON/TOML/YAML value to a string.** Conversion into schema-defined types occurs in validation, with error reporting for unsupported conversions. Preserve integer/float distinctions where possible.

### Metadata attached to each effective key

Conceptual `ConfigEntry` fields (validate syntax during Phase 0):

```text
key; value; presence; source_id; source_kind; profile; precedence_rank;
is_secret; is_default; loaded_at_ms; validation_status; content_version
```

- A public diagnostic can expose: key, type, present/missing, source kind, source name (sanitized), profile, and validity.
- A public diagnostic must **not** expose: secret content, secret lengths, plaintext digests, secret-bearing URLs, auth headers or unsafe filesystem paths.
- Provenance must not contain raw source lines or `.env` values.
- A snapshot is read-only to consumers; any builder's mutable storage must be copied/frozen before publication.
- Copies/clones must preserve `is_secret`, presence and provenance metadata.
- If a secret-marked value is interpolated into another value, the result automatically becomes secret-marked (taint propagation).
- No caller can disable secret redaction simply by changing the ordinary logging level.

### Missing, null and empty rules

| Situation | `has(key)` | Required field | Default applies? |
| --- | --- | --- | --- |
| Missing | false | failure unless default | yes |
| Explicit null | true | failure unless nullable | no (unless explicit policy) |
| Empty string | true | allowed only if schema permits | no |
| Whitespace-only string | true | default: preserve; optional `non_blank` rejects | no |
| Invalid typed string | true | validation failure | never silently |

Do not silently interpret the string literal `"null"` in `.env` as a typed null. It remains a string unless the declared schema explicitly parses it.

### Key normalization

- Canonical internal structured keys: dotted path, e.g. `database.host`, `server.port`.
- For environment variables, require an explicit field-to-env mapping (`DATABASE_HOST` ↔ `database.host`) or a documented deterministic namespace mapping (e.g. prefix `APP_`, `__` for nesting).
- Case rules must be explicit; do not assume POSIX and Windows environment-name behavior is identical.
- Reject two raw names that normalize to the same canonical key in one source unless the caller opts into a clearly documented duplicate policy.
- Escape/quote keys in diagnostics. Never treat `.` in a literal key as nesting unless the source's conversion policy explicitly says so.
- Avoid naming collisions between `APP_DATABASE__HOST` and `APP_DATABASE_HOST`; document and test the mapping.

---

## 6. Configuration sources and deterministic precedence

### Default precedence (low → high)

1. Field/schema defaults.
2. Explicit in-memory application defaults.
3. Base config file (`vault.toml`, `vault.yaml` or `vault.json` if explicitly named).
4. Profile-specific file (`vault.prod.toml`, etc.), if explicitly selected.
5. Explicit `.env` file, if explicitly loaded.
6. Explicit `.env.<profile>` file, if explicitly enabled.
7. Selected secret-provider values, for declared secret keys only.
8. Process environment for **declared/mapped keys only**.
9. Explicit runtime/test overrides passed to the builder.

The precedence is **a proposed default**, not an unavoidable hardcoded universal truth: define a `SourcePlan` with ordered source slots. Users may override order explicitly, but the final order must be documented and exposed by a safe `explain(key)` function. Never let filesystem enumeration order decide precedence.

### Additional security rules

- In **strict production mode**, forbid assigning declared secret keys from unsafe sources such as checked-in base files or source-code defaults unless explicitly authorized by schema policy.
- A value from a higher-priority source that fails validation must **fail the load**, not fall back silently to a lower-priority valid value.
- Merging a container over a scalar, or vice versa, requires an explicit policy; default to a typed conflict error.
- For object keys, default to recursive merge; for arrays, default to **replace**, not concatenate; allow explicit opt-in policies after testing.
- Empty strings are real overrides, not an instruction to remove a value.
- Support an explicit delete/tombstone operation only in memory/API overrides, not by interpreting an untrusted special string in `.env`.
- Every merge must be deterministic, including nested maps and array replacement.
- Profiles are selected by explicit constructor argument or an explicitly mapped environment key, never by guessing machine hostname or platform.
- CI and unit tests must never depend on developer workstation `.env` files; all fixtures are explicit and hermetic.

### Environment loading contract

- `nox.os.getenv` must be called **only for keys listed by schema or an explicit allowlist**.
- Treat missing process variable using `except OsError`, not by comparing against `""`.
- Do not copy `.env` values into the real process environment by default.
- Never indiscriminately scan all OS environment variables or print them in reports.
- Do not include known process secrets (`GITHUB_TOKEN`, `AWS_SECRET_ACCESS_KEY`, etc.) in test results. Use test-scoped dummy variables only.

### File lookup

- Default load is explicit path-based. Auto-discovery (e.g. ascending parent directories) is **disabled by default**.
- When auto-discovery is later enabled, bound the search root, depth and filename allowlist; never read home-directory secret files implicitly.
- Resolve paths canonically, verify allowed-root containment by directory components, and reject non-regular files, suspicious symlinks and oversized files.
- Avoid TOCTOU claims: canonicalize-then-open is not a complete defense against hostile concurrent filesystem changes. Document residual risk; use safer platform primitives only when NOX makes them available.
- No implicit writes to `.env`, JSON, YAML or TOML files when loading.

---

## 7. `.env` parser specification — pure and bounded

Implement a parser that accepts **text** and returns typed line metadata plus values (all `.env` values initially strings). Read files separately.

### Supported MVP syntax

```dotenv
# comment
APP_NAME=VaultDemo
EMPTY=
QUOTED_DOUBLE="hello world"
QUOTED_SINGLE='literal # value'
TRIMMED = value
export ENABLED=true
URL="https://example.invalid/a?x=1#fragment"
# Optional phase: interpolation references
DATABASE_URL="postgres://${DB_USER}:${DB_PASSWORD}@db.local/app"
```

**Precise behavior to document and test:**

- LF and CRLF; UTF-8 BOM only at beginning; deterministic Unicode handling.
- Empty lines and comment lines.
- `KEY=VALUE`; optional spaces around `=`; strict valid variable-name grammar (proposed: `[A-Za-z_][A-Za-z0-9_]*`).
- Optional `export KEY=...` with no side effect on process environment.
- Double-quoted and single-quoted values; delimiter/comment behavior inside quotes.
- Unquoted inline `#` comment policy must be explicitly specified and tested; do **not** blindly remove everything after the first `#`.
- Double-quoted escape sequences: specify exactly `\n`, `\r`, `\t`, `\\`, `\"` (and reject or retain unsupported escapes by documented strict policy).
- Single quotes: literal contents by default; no shell evaluation.
- Missing key/equals sign, invalid names, malformed quotes, duplicate assignments and embedded NUL: default **parse error** with safe line/column numbers.
- File with a trailing newline must not generate a phantom key.
- Preserve `""` as an explicit empty string; do not confuse with absent.
- Multiline quoted values: **P1 only**, after syntax/limits are defined. Do not claim support in MVP.
- Variable interpolation happens **after parsing and merging**, not as file-order side effects.

### Explicitly forbidden syntax

- `$(...)`, backticks, shell commands, process substitution, glob expansion, arbitrary expressions and execution of imported files.
- Shell `source` semantics, arbitrary `include` paths, command or subprocess execution.
- Password values appearing in error messages for malformed lines.
- Expanding a variable by implicitly reading additional OS environment keys not in the allowlist.

### Parser limits (initial suggested defaults; configurable below hard caps)

| Limit | Initial default | Rationale |
| --- | ---: | --- |
| Single file size | 1 MiB | avoid whole-file exhaustion |
| Number of keys/source | 2,000 | bounded indexing/merging |
| Single raw value | 64 KiB | prevent giant secret/report values |
| Key length | 128 bytes | prevent parser abuse |
| Number of sources/load | 32 | prevent unbounded startup work |
| Include/lookup depth | 0 in MVP | no implicit recursive file loads |
| Interpolation depth | 16 | cycle/expansion protection |
| Expanded value size | 128 KiB | avoid exponential expansion |

Reject values exceeding a limit with a safe error code; do not truncate silently. Configurable limits must still obey documented hard ceilings.

### Reference parser test vectors

- Empty file, comment-only file, BOM, CRLF, duplicate keys, whitespace around `=`, `export`, single/double quotes.
- Literal `#` inside quotes, literal `=` in values, URLs containing `#` or `=`.
- `KEY`, `=VALUE`, malformed quote, Unicode key vs Unicode value, NUL byte, overlong line.
- Repeated key with same value and with different value; deterministic strict failure.
- Secret value on malformed line: error exposes only file identifier/line/column/key, never parsed value.
- Fuzzer supplies invalid bytes and random quote/escape combinations without crash, hang or OOM.

---

## 8. Typed schema validation

### Required field types

- `string`, `int`, `float`, `bool`, `string_list`, `object`/nested schema.
- `duration_ms` and `url` may be built from validators after core typing is complete.
- A separate `secret: true` flag is supported on any type.

### Proposed field declaration contract (illustrative, NOT yet a verified NOX API)

```text
ConfigSchema schema
  required_string("app.name")
  optional_int("server.port", default=8080, min=1, max=65535)
  required_string("database.url", secret=true, non_blank=true)
  optional_bool("features.verbose", default=false)
  optional_string_list("cors.origins")

ConfigBuilder builder
  add_file("config/vault.toml")
  add_dotenv(".env")
  add_process_env(allowed_keys_from_schema)
  set_schema(schema)
  build_and_validate() -> ConfigSnapshot | safe error report
```

The exact NOX signatures are a Phase 0 decision. Avoid creating a fluent DSL that requires unverified decorators, runtime reflection, keyword default syntax or unsupported overloading.

### Conversion policy

- `int`: strict signed decimal strings from `.env` with overflow detection; no silent floating-point truncation.
- `float`: strict numeric syntax, reject NaN/Infinity by default for application settings.
- `bool`: accept only documented case-insensitive variants such as `true/false`, `1/0`, `yes/no`, `on/off`, if the schema opts into the broader grammar. Default strict mode may use only `true/false`.
- `string_list`: explicit format; recommended use native arrays from structured files and an optional delimiter policy for `.env`. Don't silently split strings containing commas inside quoted values.
- `duration`: require units (`ms`, `s`, `m`, `h`) if added; safe bounds and overflow checks.
- `URL`: parse through verified stdlib URL support, enforce scheme/host constraints when required. Never include username/password in diagnostics.
- Avoid converting a structured array/object into arbitrary textual representation that changes semantics.

### Constraint system

Support field-level `required`, `nullable`, `non_blank`, `min`, `max`, `min_length`, `max_length`, `one_of`, and custom *pure* validator callbacks where verified. Add cross-field validation in a later phase (`TLS_ENABLED` requires cert/key; production requires strong secret settings).

### Validation result

Validation should be **aggregated**: return every safe, independently discoverable error instead of only the first error. Errors contain:

```text
code | canonical_key | source_kind | sanitized_origin | expected_type |
constraint_id | optional line/column | severity
```

They must not contain raw invalid values or values embedded in exception strings. Stable error codes should include:

`CONFIG_MISSING`, `CONFIG_NULL_NOT_ALLOWED`, `CONFIG_EMPTY_NOT_ALLOWED`, `CONFIG_TYPE_MISMATCH`, `CONFIG_OUT_OF_RANGE`, `CONFIG_DUPLICATE_KEY`, `CONFIG_SOURCE_UNAVAILABLE`, `CONFIG_PARSE_ERROR`, `CONFIG_INTERPOLATION_CYCLE`, `CONFIG_SECRET_SOURCE_DENIED`, `CONFIG_PATH_OUTSIDE_ROOT`, `CONFIG_SIZE_LIMIT`, `CONFIG_PROVIDER_UNAVAILABLE`, `CONFIG_SCHEMA_VERSION_MISMATCH`.

Errors should sort consistently (source rank, key, code) so tests and CI are deterministic.

---

## 9. Interpolation — deterministic, not executable

### Proposed forms

- `${KEY}` — required reference; missing key is an error in strict mode.
- `${KEY:-literal}` — optional future explicit fallback, **literal only**, no nested expression evaluation in MVP.
- `$$` — optional escaping, documented before introduction.

### Resolution invariants

1. Resolve **after** source precedence, not within individual source file order.
2. References can access only keys that exist in the merged snapshot; accessing process environment implicitly is forbidden.
3. Build a dependency graph / DFS visitation state; detect direct/indirect cycles and report affected keys without values.
4. Bound depth, number of references per value and expanded output length.
5. Preserve exact Unicode and escape semantics; avoid recursive regex substitution without termination proof.
6. A referenced secret propagates secret classification to the final derived key. Fail if a secret would flow into an explicitly public-only destination under strict policy.
7. Never expand a secret into a URL that the library will later log or send without the caller explicitly acknowledging the destination.
8. No `$(command)`, Python expressions, arithmetic execution, template imports, implicit HTTP calls or disk reads.
9. Test `A -> B -> C`, `A -> A`, `A -> B -> A`, missing references, empty values, repeated references and size bombs.
10. If the resolution graph fails, return no new published snapshot.

---

## 10. Secret classification, redaction and access

### Threat model

Assume local files may be accidentally committed, users may print diagnostics, untrusted config files may contain crafted paths, environment sources may contain secrets, remote services may fail or return hostile content, and logs/crash artifacts may be retained by third parties. VAULT cannot prevent privileged debuggers or process memory dumps from reading secrets; document this honestly.

### Secret API principles

- Secret values must use a dedicated `SecretValue`/`SecretRef` abstraction or equivalent verified NOX wrapper with **explicit access** to the underlying value.
- The default textual rendering of a secret wrapper (if NOX rendering hooks exist) must be `[REDACTED]`, not its contents. Test every relevant conversion/printing path.
- `to_json`, diagnostics, `explain`, snapshots, logging and debug views must redact secrets by metadata, not just by matching names.
- Secret metadata propagates across merges, cloning, interpolation and derived fields.
- The API should distinguish `get_secret(key)` (explicit value access) from `describe(key)` (safe metadata). Do not provide a general `dump_all_plaintext()` convenience API.
- No secret previews, secret lengths or plaintext fingerprints in ordinary diagnostics.
- Avoid implementing a `reveal` API that logs/audits its **returned value**.
- Do not write secrets to a temp file for parsing, formatting, caching or testing.
- When a required secret is missing, report its **key name** and reason, never a fallback value.
- Process environment is not an encrypted secret store. It may be inherited by children; document this limitation.
- Zeroization guarantees for native NOX strings are **not assumed**. Use best-effort short lifetimes, minimal copies and no persistence; do not advertise guaranteed memory erasure.

### Safe redaction strategy

1. Prefer rendering structured config entries one-by-one and replacing secret-marked values with `[REDACTED]`.
2. Redact known secret key names in `KEY=value` formatted messages as a secondary defense.
3. Optionally use `nox.log.redact(message, known_secrets)` as *defense in depth*, not a substitute for structured redaction.
4. Exclude bearer tokens, cookies, auth headers and `.env` raw lines from exception payloads at their source.
5. Treat secret-containing derived values as secrets; never downgrade classification on override without an explicit policy.
6. Test URLs with passwords, JSON escaping, multi-line tokens, repeated values, small and large secrets, Unicode and nested JSON.

### Forbidden implementations

- “Encryption” using base64, XOR, SHA-256 or simple obfuscation.
- Hardcoded demo passwords in runtime sources (dummy fixtures may use obvious test-only placeholders).
- Logs containing the loaded `.env` text.
- CLI `vault explain --show-secrets` as a default or casually accessible command.
- Automatic publication of secret data to CI artifacts or package registry.

---

## 11. Secret providers — optional and isolated

### Provider interface (conceptual)

```text
SecretProvider
  id: non-sensitive string
  get(key: str) -> SecretFetchResult
  health() -> ProviderHealth
  close() -> None

SecretFetchResult
  found: bool
  value: SecretValue or equivalent
  version_token: opaque, optional, never secret-bearing
  expires_at_ms: optional
  source_id: sanitized
  error_code: safe and stable
```

Verify whether NOX protocols, class-based interfaces or function callbacks are the most compatible representation. Providers must be injectible for deterministic unit tests.

### Provider implementations

**Phase 1 / 2:**

- `EnvSecretProvider`: reads explicitly configured names with `nox.os.getenv`.
- `FileSecretProvider`: reads allowlisted, canonically confined, bounded files with clear permissions guidance; does not assert POSIX permission APIs are available when unverified.
- `MemorySecretProvider`: test-only fake; never use it as a recommended production secret store.

**Later optional adapters:**

- `HashicorpVaultProvider`: verified HTTPS endpoint, auth strategy, namespace/mount/key mapping, renewal policy, token handling, versioned keys, timeout/retry limits. Never implement an unverified Vault API and call it production-ready.
- `CloudSecretsProvider`: AWS Secrets Manager, Google Secret Manager, Azure Key Vault **as separate opt-in adapters**, each gated by available SDK or a narrowly scoped authenticated HTTPS implementation.
- A provider for app-specific encrypted stores only after an audited encryption/KMS dependency and threat model are approved.

### Remote provider requirements

1. Constructor requires explicit provider kind, endpoint, namespace/path mapping and credential source.
2. HTTPS only; no redirect to an unapproved host; prevent SSRF and cloud metadata endpoint access.
3. Validate canonical hostname, scheme and port against an allowlist before every request; do not weaken TLS validation.
4. Bounded connect/read timeouts, response size, retry count and backoff. No unbounded retry loop.
5. Fail closed on invalid auth, missing required secret, malformed payload or unsafe endpoint.
6. Do not log tokens, request/response bodies, headers, secret IDs that themselves contain credentials, or raw provider exceptions.
7. Never cache plaintext secrets on disk by default. If memory caching is enabled, define TTL, explicit refresh and invalidation.
8. Rotation is resolved through a new validated snapshot; old snapshots remain usable only per documented grace policy, and memory retention limitations are explicit.
9. No implicit provider discovery based on environment variable prefixes or cloud metadata.
10. Test with an in-process mock provider; real external-provider CI uses dedicated credentials and opt-in secrets without exposing them.

### Provider failure modes

Support explicit `fail_closed` (default), `optional` (only for non-required keys), and `last_known_good` (reload only, never cold start of required secrets without approved policy). Never silently substitute a default password when production provider is unavailable.

---

## 12. Profiles, snapshot reload and operational behavior

### Profiles

- Supported conceptual names: `development`, `test`, `staging`, `production`; allow custom sanitized profile names.
- Profile names select explicit filenames and sections; no automatic cross-environment fallback that could load production credentials in development.
- `production` enables strict checks: required secrets, no default sensitive values, no plaintext secret serialization and stricter file source restrictions.
- `.env.test` fixtures must use dummy values and a hermetic root.
- Unknown or misspelled profile is an error, not a fallback to development.

### Reload

- No watcher, polling or background reload until a reliable NOX scheduling and filesystem notification mechanism is verified.
- `reload()` builds a candidate snapshot through all ordered sources, interpolation and schema validation.
- On any failure: return a structured report; retain the **entire previous snapshot**; no partial update.
- On success: publish a new versioned immutable snapshot; future reads use it, in-flight reads complete on their old snapshot.
- Consider concurrent readers and object lifetimes. Atomic pointer swaps are prohibited until the exact safe ownership primitive is verified in NOX.
- When watchers are eventually introduced, debounce, cap refresh frequency and make stop/shutdown deterministic.
- Provider credential rotation and file reload must not expose half-updated settings or accidentally duplicate background tasks.

### Operational metrics

Allowed: load success/failure, source-kind counts, validation error counts by stable code, reload duration, cache hit/miss, provider availability. Not allowed: secret values, plaintext hashes, high-cardinality keys containing sensitive data, raw URLs with credentials.

---

## 13. Proposed public API and UX contract

The following is an **API design contract**, not promised compilable syntax. Agents must implement an equivalent NOX-idiomatic API after Phase 0 probes.

### Primary types and operations

```text
VaultOptions: strict, profile, allowed_root, limits, duplicate_policy,
              precedence_plan, allow_process_env, allow_remote_secrets

ConfigBuilder:
  with_defaults(...)
  with_file(path, format)
  with_dotenv(path, optional=false)
  with_process_env(keys_or_schema_mapping)
  with_profile(name)
  with_secret_provider(provider)
  with_overrides(...)
  with_schema(schema)
  build() -> ConfigSnapshot | ConfigLoadError

ConfigSnapshot:
  has(key) -> bool
  is_null(key) -> bool
  get_string(key) -> str
  get_int(key) -> int
  get_float(key) -> float
  get_bool(key) -> bool
  get_list(key) -> typed/structured list
  get_secret(key) -> explicit secret access
  explain(key) -> SafeOriginDescription
  redacted_json() -> str
  version() -> str/int

ValidationReport:
  valid() -> bool
  error_count() -> int
  errors() -> list[SafeConfigDiagnostic]
```

All getters must define missing/null/type-mismatch behavior consistently. Prefer throwing typed exceptions or returning result objects *according to verified compiler capabilities*. Avoid mixing `""`, `0`, `False` and `None` as undifferentiated failure sentinels.

### Illustrative ordinary usage (pseudocode)

```text
schema = Schema()
schema.add_string("app.name", required=true)
schema.add_int("server.port", default=8080, min=1, max=65535)
schema.add_string("database.url", required=true, secret=true)

config = ConfigBuilder()
  .with_file("config/vault.toml", format="toml")
  .with_dotenv(".env", optional=true)
  .with_process_env(schema.env_names())
  .with_schema(schema)
  .build()

port = config.get_int("server.port")
origin = config.explain("database.url")  # NO secret value
```

The final documented quickstart must use real compiling `.nox` code. Do not copy pseudocode directly into production examples.

### Optional CLI contract

- `vault validate --config <path> --profile <name>`: nonzero on invalid config; safe error codes.
- `vault explain <key> --config <path>`: safe origin metadata only.
- `vault check-env --schema <path>`: lists missing required **names**, never secret values.
- `vault version`: package/compiler version information.
- `vault --help`: no config loading or network access.
- CLI arguments must be parsed without building shell command strings. The CLI is optional; the library must work when CLI is not installed.
- No `printenv`, dump-all-secret, decrypt-secret or interactive credential prompts in the v1.0 CLI.

---

## 14. Integration strategy across the NOX ecosystem

The core must work with plain NOX applications without any framework dependency. Adapters are **thin mappings** from validated config to consumer-specific initialization data:

| Consumer | Optional adapter goal | Forbidden coupling |
| --- | --- | --- |
| Aether | HTTP server bind address, port, TLS/proxy settings | importing Aether from VAULT core |
| Nyx | route/service startup settings | booting a web app just to load config |
| Noxen | environment-aware app setup | global settings mutation |
| Nova | service/client settings | embedding cloud secrets in generated code |
| Feather | DB URL, pool sizes, migration policy | exposing DB password in errors |
| Atlas | data-source paths and reader options | reading arbitrary files outside allowed root |
| Quartz | safe test configuration fixtures and assertions | making Quartz a runtime requirement |
| Aura | frontend-safe **public** config export only | shipping server-side secrets into browser/mobile bundles |
| Pulse/Ember | optional metrics/cache provider settings | secret-bearing metrics or unbounded cache retention |

Do not make assumptions about any consumer's actual current API. Read that repository and write an adapter compatibility test before declaring integration complete. Browser/mobile targets must **never** receive a server-side secrets snapshot; define an explicit public-config allowlist.

---

## 15. Security requirements and abuse cases

### File handling

- Canonical allowed-root confinement using path components, not only a raw string prefix.
- Reject traversal (`../`), symlink escape, unknown file types and excessive file sizes.
- Avoid wildcard paths and recursive file discovery by default.
- File read is read-only; do not overwrite on failed parsing or validation.
- Handle Windows drive letters, UNC paths, path separators and CRLF in CI.
- Permission checks must be implemented only after verifying NOX exposes required metadata; otherwise provide documented deployment guidance without claiming enforcement.

### Parser and interpolation

- Reject NUL and malformed escapes.
- Bound source and reference graph sizes; cycles fail with a safe diagnostic.
- Ensure `${DATABASE_PASSWORD}` into another field preserves secret metadata.
- Never use format strings to execute expressions.
- Reject duplicate canonical names unless an explicit duplicate policy is selected.
- Reject unsupported YAML/TOML constructs with source-level context but no raw contents.

### Remote provider

- No network unless explicitly enabled.
- HTTPS and certificate validation required; no insecure TLS bypass switch in normal API.
- Endpoint host/port allowlist; DNS rebinding and redirects addressed in threat model.
- Credentials read only from explicitly named sources; no implicit cloud metadata crawl.
- Timeouts, retries, backoff, response-size limits and connection lifecycle tested.
- Transient failure must never silently result in an empty secret.

### Error handling and observability

- Never include values in `str(exception)` or `to_json()` for errors.
- Preserve enough safe context for diagnosis: key, source, line, code, expected type.
- Distinguish load failure from validation failure from provider failure.
- Fail on missing required production secrets before the app starts listening.
- Test logging at DEBUG level and make sure no secrets appear.
- Tests scan generated reports and stdout/stderr for sentinel secret strings.

### Threat-model test scenarios

1. A checked-in `.env` contains a dummy secret and the CLI should redact it.
2. A crafted source path escapes the allowed root via `..` and via symlink.
3. A massive file is rejected before full loading.
4. An interpolation bomb or cycle is rejected quickly.
5. Two sources supply the same key with different types; precedence and validation remain deterministic.
6. A required secret is absent in production; startup fails closed.
7. A provider returns a secret with line breaks, quotes, Unicode and JSON metacharacters; no log leaks it.
8. A provider URL redirects to loopback, metadata IP or unapproved hostname; block before sending credentials.
9. A reload changes one valid key but includes another invalid key; the published snapshot remains wholly unchanged.
10. Two differently cased environment names collide under Windows semantics; defined conflict behavior, not nondeterministic priority.
11. A file reader fails midway or a provider times out; cleanup occurs without partial config publication.
12. A malicious config key attempts to override a required secret through an untrusted file in strict production mode; reject it.

---

## 16. Test strategy — Quartz optional, real execution mandatory

### Mandatory test categories

| Area | Minimum evidence |
| --- | --- |
| Dotenv parsing | positive, invalid grammar, BOM, CRLF, Unicode, comments, duplicates, limits |
| Process env | present vs missing vs empty, explicit allowlist, no mutation |
| Merge | 3+ sources, nested maps, arrays, types, deterministic precedence |
| Schema | required, null, empty, bounds, invalid bool/int/float, aggregate diagnostics |
| Interpolation | multi-hop, cycle, unknown reference, expanded-length cap, secret taint |
| JSON/TOML/YAML | real stdlib parsing, nested values, unsupported features, bad documents |
| Secrets | redacted `explain`, redacted errors, no stdout/stderr leakage |
| Providers | fake success, miss, timeout, bad auth, retry exhaustion, no implicit network |
| Profiles | correct precedence, strict prod policy, unknown profile error |
| Reload | successful atomic publication, failure rollback, stable reader snapshot |
| Security | traversal, symlink escape, oversized file, config injection, SSRF denial |
| Package | clean import, package installation, locked dependency reproduction |
| Backends | type check, execution, QBE/LLVM parity where supported |
| Cross-platform | Windows paths/env case/CRLF, Linux/macOS symlinks/files |

### Test design rules

1. Every `.nox` test file exits nonzero (or raises) on failure; no optional `print("PASS")` without assertions.
2. Unit tests run without network, cloud credentials or real production `.env` files.
3. Use dummy secrets that are obvious fixture placeholders. Verify by scanning logs/reports that sentinel values do not appear.
4. Use in-memory sources and fake providers for determinism; separate opt-in integration jobs for real providers.
5. Property/fuzz tests must be bounded and reproducible. If Quartz is available, record seed/case index and minimized counterexample; otherwise use an explicit deterministic runner.
6. Test invalid configurations against all supported formats rather than only `.env`.
7. Test both **expected success** and **expected rejection**. An unsupported path must fail safely.
8. Fail if a test was skipped due to a missing mandatory platform capability; optional-provider tests must report explicit SKIP and reason.
9. Do not rely on a local user's env, current directory or git-ignored files: set up temporary fixture roots explicitly.
10. Do not introduce flaky wall-clock/reload/remote tests into the default suite. Use fake clocks or generous controlled deadlines where necessary.
11. CI must independently confirm the artifact being validated belongs to the exact git commit/tag under test.
12. Maintain regression tests for every reported vulnerability and each bug found during development.

### Explicit acceptance table: canonical precedence

Given `default=1`, `base_file=2`, `profile=3`, `.env=4`, `.env.profile=5`, `provider=6`, `process_env=7`, `override=8`, final value must be `8` with source `override`, **unless policy intentionally rejects that source for the declared key**. Remove the winner in successive tests to verify the exact next source. Test invalid winning source: validation failure, **not** fallback.

### Explicit acceptance table: secret redaction

For a secret sentinel such as `VAULT_TEST_SECRET_DO_NOT_PRINT_9E2D`, assert that it is absent from:

- `explain(key)` and all diagnostic strings;
- `redacted_json()` and CLI output;
- error `str()`, `to_json()`, test reports and debug logs;
- snapshots, diffs, provider health/status and metrics.

The legitimate explicit `get_secret(key)` path must still retrieve the exact secret for an authorized caller; it must not be silently replaced by `[REDACTED]` in application use.

### Compiler/backend matrix

- `noxc check vault.nox`
- `noxc test`
- Explicit QBE backend runtime smoke tests (verified correct command syntax)
- Explicit LLVM backend runtime smoke tests where available
- Windows x64, Linux x64, Linux ARM64, macOS ARM64
- Hosted with filesystem + process capabilities; pure parsing components separately assessed for restricted profiles

Do not claim broad platform support until the test matrix executes successfully.

---

## 17. CI, packaging and release engineering

### CI gates

- Checkout exact commit, install pinned NOX compiler, verify `noxc version`.
- `noxc check vault.nox` must pass.
- `noxc test` must run every mandatory test file and produce nonzero on a failure.
- QBE and LLVM smoke tests executed on supported platforms.
- Validate `.env.example` contains dummy values only; scan repository for accidental credential patterns and known test sentinel leaks.
- Check `nox.lock` consistency and dependency provenance.
- Ensure no unexpected network requests or filesystem mutations during pure tests.
- Optional remote-provider jobs are isolated, require explicit CI secrets and are **never** part of the mandatory green matrix unless configured.
- Upload only safe, redacted test artifacts. Enforce a minimum coverage of edge/negative cases qualitatively; do not invent percentage coverage without instrumentation.
- Gate registry publication on **successful mandatory CI for the exact commit**, not an unrelated older run.
- Avoid invalid GitHub Actions expression contexts: `runner.os` is appropriate in step-level contexts; for job-level decisions prefer supported `matrix` context.

### Package metadata

- Project `name`: `vault` if registry availability and naming rules permit; otherwise choose and document a non-conflicting alias.
- Public facade: `vault.nox` with explicit imports and version information.
- Use `nox.json`/`nox.lock` conventions from the current NOX toolchain.
- Package dependencies: none at runtime for MVP beyond NOX stdlib; optional integrations must not force unnecessary package dependencies.
- A `bin` entry is optional and added only once a real CLI exists.
- Tag releases with SemVer; do not move released tags after publication. A correction should receive a new patch version.
- `noxc publish` submits package **metadata** to a moderated registry; do not say a package is publicly available until acceptance is confirmed.
- Document supported compiler and backend versions in each release.

### Release checklist

1. All mandatory CI jobs green on the exact release SHA.
2. `README` quickstart examples actually compile and run.
3. No uncontrolled secrets in repo, test logs or release artifacts.
4. Compatibility matrix updated and optional integrations clearly marked.
5. Changelog describes only tested, implemented behavior.
6. Any migration/breaking change has a migration guide.
7. New tag created from green commit; never rewrite the prior stable tag.
8. Registry approval status stated accurately.

---

## 18. Roadmap, dependencies and strict phase acceptance gates

**Agents must execute the phases in order.** Features within a phase may be implemented incrementally, but no phase is declared complete until all acceptance criteria pass. The phase names below are target milestones, not existing releases.

### Phase 0 — Capability audit & package skeleton

**Build:** `nox.json`, `vault.nox`, `AGENTS.md`, starter README, `.gitignore`, test harness, `docs/compatibility.md`; compile probes for core types, exceptions, env, filesystem, JSON/TOML/YAML and provider interfaces.

**Gate:** `noxc check vault.nox` passes; `noxc test` runs at least one real assertion file; required NOX API signatures are documented with upstream links; no runtime third-party dependencies; exact compiler version pinned.

### Phase 1 — Pure `.env` parser

**Build:** `env/parser.nox`, token/value representation, safe syntax errors, duplicate policy, limits, BOM/CRLF handling, escaped quoted strings, parser fixtures.

**Gate:** full positive/negative parser matrix; no process environment mutation; no network; oversized/malformed cases reject safely; all error messages redacted; parser fuzz budget bounded.

### Phase 2 — Core snapshots, sources & precedence

**Build:** `core/types`, `snapshot`, `merge`, `sources/memory`, `env/source`, `env/process`; deterministic source plan; missing/null/empty; provenance.

**Gate:** winner/provenance deterministic under all source orders; no OS env enumeration; explicit allowlisted env reads; invalid winning value cannot silently fall back; published snapshots are immutable to users.

### Phase 3 — Typed schema and validation

**Build:** field types, defaults, required/nullable/non-blank, ranges, strict coercion, aggregate safe diagnostics and typed getters.

**Gate:** all type/boundary tests pass; production missing-secret fails closed; multiple errors are reported in stable order; no secret material appears in failures.

### Phase 4 — JSON, TOML, YAML bridges

**Build:** type-preserving conversion through real NOX stdlib parsers, explicit structured-file source names, root-constrained loading.

**Gate:** nested objects and arrays preserved; null/empty distinctions tested; malformed/unsupported documents rejected; size/path limits enforced; round-trip behavior not claimed beyond real guarantees.

### Phase 5 — Profiles, interpolation & secret taint

**Build:** base/profile overlays, explicit profile selection, graph-based `${KEY}` expansion, cycle detection, secret classification propagation.

**Gate:** 3-level reference chains, missing/cycle/expansion bomb tests pass; secret-derived fields remain secret; no file-order-dependent expansion; strict production source policy enforced.

### Phase 6 — Redaction & safe inspection

**Build:** secret wrappers/access policy, `explain`, safe JSON summary, diagnostic rendering, documented threat model.

**Gate:** intentional secret-value retrieval works only through explicit API; secret sentinels absent from every automatic output; DEBUG logging test clean; no claimed encryption or zeroization guarantee without implementation.

### Phase 7 — Local secret provider adapters

**Build:** provider contract, fake provider, named environment provider, root-restricted file provider, safe provider errors.

**Gate:** provider injection/selection explicit; missing/invalid secrets fail closed; file traversal and size protections pass; no secret logged; no hidden cloud/network access.

### Phase 8 — Reload & last-known-good snapshots

**Build:** manual reload, validated candidate builds, versioned snapshot publication, explicit failure results.

**Gate:** invalid reload leaves old snapshot intact; valid reload publishes complete new snapshot; no partial state, use-after-free or duplicate background jobs; concurrency primitives confirmed on actual NOX backends.

### Phase 9 — Optional remote provider framework

**Build:** HTTPS provider abstraction, endpoint allowlists, timeouts, retries, bounded responses, safe auth/source injection, in-process fake transport. HashiCorp/cloud adapters separately scoped.

**Gate:** local/fake integration tests prove auth error, timeout, redirect block, SSRF block, rotation and cache expiry; no remote traffic in default tests; zero leaked tokens/secret bodies. Do **not** mark specific cloud integrations ready without verified live behavior.

### Phase 10 — Optional CLI and developer ergonomics

**Build:** `validate`, `check-env`, `explain`, version/help, clear exit codes, examples and docs.

**Gate:** CLI execution (not merely type check) verified; invalid config returns nonzero; stdout/stderr sentinel scan clean; `--help` never loads secrets; no secret-revealing default flags.

### Phase 11 — Optional ecosystem adapters

**Build:** individually approved Aether, Nyx, Feather, Noxen, Nova, Aura-public-config-only and Quartz test helpers.

**Gate:** each adapter has a real integration test against the versioned consumer; core import remains framework-independent; browser/mobile export contains explicitly allowlisted non-secret config only.

### Phase 12 — v1.0 hardening and release

**Build:** finalized APIs, stable error codes, benchmarks, compatibility matrix, full security review, examples, migration guide, release automation.

**Gate:** four-platform green matrix, supported backend runs, mandatory tests all executed, no secret exposure, provider opt-in verified, approved registry state correctly reported, no advertised feature remains a mock or unexecuted TODO.

### Prioritization when time is limited

**MVP = Phases 0–3** (`.env` + env + deterministic merges + typed validation).  
**Practical config package = Phases 0–6** (JSON/TOML/YAML + profiles + redaction).  
**v1.0 target = Phases 0–8 plus release hardening**, with remote integrations and optional CLI/adapters separately versioned if they are not complete.  
Do **not** delay a secure, coherent stable core solely to implement every external cloud provider; state optional functionality honestly.

---

## 19. Mandatory regression and acceptance scenarios

Agents must maintain an explicit test matrix covering at least these scenarios:

| ID | Scenario | Expected outcome |
| --- | --- | --- |
| ENV-01 | Missing process env key | `OsError` internally mapped to missing, not empty |
| ENV-02 | Empty but present process env key | explicitly empty, not missing |
| ENV-03 | `.env` quoted `#` and `=` | exact value preserved |
| ENV-04 | `.env` duplicate canonical key | strict deterministic parse failure |
| ENV-05 | `.env` `$(cmd)` | treated as literal or explicitly rejected; never run |
| ENV-06 | `.env` invalid quotes | safe line/column error, no raw value leak |
| ENV-07 | CRLF + BOM | valid parse with documented normalization |
| ENV-08 | Oversize config | rejects before unbounded allocation |
| MERGE-01 | Multi-source precedence | exact winning source and value |
| MERGE-02 | Invalid high-priority override | validation failure, no fallback |
| MERGE-03 | Nested object overlay | recursive merge, array replace |
| MERGE-04 | Missing/null/empty | remain distinct |
| MERGE-05 | Duplicate normalized env mapping | explicit collision error |
| SCHEMA-01 | Required secret missing | fail closed, no guessed fallback |
| SCHEMA-02 | `int` overflow | validation failure |
| SCHEMA-03 | Invalid boolean | validation failure |
| SCHEMA-04 | `NULL` on nonnullable | validation failure |
| SCHEMA-05 | Multiple invalid fields | aggregate stable safe diagnostics |
| INTERP-01 | `A -> B -> C` | deterministic resolved value |
| INTERP-02 | Direct/indirect cycle | reject with safe cycle code |
| INTERP-03 | Expansion bomb | reject at depth/size limit |
| INTERP-04 | Secret reference | derived field remains secret |
| SECRET-01 | Debug logging | secret sentinel absent |
| SECRET-02 | Redacted JSON/explain | secret sentinel absent |
| SECRET-03 | Explicit `get_secret` | exact authorized value available |
| SECRET-04 | Missing provider secret | no empty/default production substitute |
| FILE-01 | Path traversal | reject outside allowed root |
| FILE-02 | Symlink escape | reject outside allowed root |
| FILE-03 | Wrong path prefix | `/app/config-evil` rejected for `/app/config` |
| FILE-04 | Missing optional file | explicit skip; required file fails |
| FILE-05 | Changed file during load | fail safely; no partial publish |
| FORMAT-01 | JSON/TOML/YAML nested values | preserve typed structure |
| FORMAT-02 | Invalid syntax | safe failure, no silent partial parsing |
| PROFILE-01 | `production` strict | rejects insecure secret sources |
| PROFILE-02 | Unknown profile | error, no silent dev fallback |
| PROVIDER-01 | Remote provider disabled | zero network calls |
| PROVIDER-02 | Host redirect off allowlist | reject before forwarding credentials |
| PROVIDER-03 | Timeout/exhausted retry | bounded failure |
| PROVIDER-04 | Rotation | validated new snapshot, no partial update |
| RELOAD-01 | Invalid changed config | retain last-known-good snapshot |
| RELOAD-02 | Valid changed config | publish fully validated new snapshot |
| CLI-01 | `validate` with bad config | nonzero exit and redacted error |
| CLI-02 | `--help` | no file/env/provider access |
| CI-01 | Four platform test matrix | actual runtime tests complete |
| CI-02 | QBE/LLVM parity | supported backends match semantics |
| CI-03 | Secrets scan | no test sentinel in any artifact |
| CI-04 | Publish gate | exact release commit green before submit |

---

## 20. Code review checklist

Every PR must answer **YES** to all applicable checks:

**Correctness**
- [ ] Does it compile against the pinned NOX version, not only hypothetical syntax?
- [ ] Is the new behavior tested with a real executing test file?
- [ ] Are missing, empty, null, invalid and duplicate cases distinguishable?
- [ ] Is source precedence deterministic and provenance correct?
- [ ] Does a malformed higher-priority value fail closed?
- [ ] Is parsing bounded and exception-safe?
- [ ] Are backend/platform differences documented?

**Security**
- [ ] Can any secret appear in exceptions, logs, `to_json`, test reports or CLI output?
- [ ] Are untrusted paths canonicalized and contained by path segments?
- [ ] Does this operation require network or filesystem access that was not explicitly granted?
- [ ] Does interpolation propagate the secret classification?
- [ ] Are provider calls authenticated, timeout-bounded, allowlisted and non-logging?
- [ ] Does this avoid code execution, shells and unsafe evaluation?
- [ ] Does production mode reject missing/weak secret sources?

**Integration and operability**
- [ ] Can `vault.nox` still import with no optional frameworks installed?
- [ ] Does failure leave the currently published snapshot unchanged?
- [ ] Are resources, handles, tasks and cache entries cleaned up?
- [ ] Are old configuration artifacts either compatible or explicitly rejected?
- [ ] Are docs/examples accurate and compilable?
- [ ] Does CI actually execute tests on the exact commit being released?

---

## 21. Agent task output format

For each completed task, respond with a concise engineering report using exactly these sections:

```text
IMPLEMENTED
- Modules/files actually changed and APIs actually implemented.

VERIFIED
- Exact compiler version and commands executed.
- Test files passed, negative tests executed, platforms/backends verified.

SECURITY
- Redaction/no-network/path constraints checked.
- New trust boundaries, if any.

NOT VERIFIED / KNOWN RISKS
- Missing backend/provider/platform coverage; never hide this section.

NEXT STEP
- Smallest remaining task in the current phase and its acceptance criteria.
```

### Stop-the-line conditions

**Stop and fix before further features** if any of the following occurs:

- CI cannot start, tests fail, or a new feature only passes static type checking.
- A secret can reach an automatic log/report/error message.
- A required production secret silently falls back to a default or invalid value.
- A provider sends credentials to an unapproved host or follows an unsafe redirect.
- The loader reads arbitrary files outside the declared root.
- A reload publishes any partial or unvalidated state.
- The implementation depends on a NOX API that is absent from the pinned compiler.
- A release tag references an unverified or failing commit.

---

## 22. Start here — first autonomous implementation assignment

**Assignment:** Implement **Phase 0**, then Phase 1 only after the capability gate passes.

1. Examine the actual repository; do not overwrite existing files or assume it is empty.
2. Create the minimum package structure, README, `nox.json`, public `vault.nox` facade and test harness.
3. Pin the compiler and capture current NOX behavior for `nox.os.getenv`, `nox.fs.metadata`, `nox.path.canonicalize`, `nox.json`, `nox.toml` and `nox.yaml`.
4. Write a compiling test that imports the public package and uses at least one real assertion.
5. Implement a pure dotenv parser slice: `KEY=value`, comment, blank, `export`, quoted values, missing/empty distinction, duplicate detection and CRLF.
6. Add safe diagnostics and hard input limits **before** adding real file/environment loaders.
7. Test malicious input and secret sentinel redaction.
8. Run checks and real tests and report results. If a compiler API is missing, record the incompatibility and adapt the plan rather than fabricating an API.
9. Do not implement remote providers, hot reload, a framework adapter or a flashy CLI during Phase 1.

**Definition of done:** A standalone, NOX-native package compiles, its dotenv parser can correctly read a documented grammar without executing anything, invalid inputs fail safely, and CI executes the test suite rather than relying on type-check-only fallbacks.

---

## Final engineering principle

> **VAULT must make configuration predictable and secrets difficult to leak.**  
> A smaller library that fails closed, explains provenance safely and passes real tests is better than a large configuration platform that hides invalid values, silently changes the process environment or claims security properties it cannot guarantee.
