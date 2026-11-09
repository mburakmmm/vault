# VAULT v1.0.1 Patch Release — TODO & Remediation Matrix

Tracked remediation plan addressing all architectural, security, and correctness findings from the v1.0.0 independent review.

---

## 🚨 P0: Critical Security & Confinement Vulnerabilities

- [x] **SEC-01: Linux & Unix Root Confinement Case-Sensitivity**
  - **File:** `sources/files.nox`
  - **Issue:** `check_path_confinement()` uses `eq_ignore_case()` on all platforms. On Linux/Unix, `/data/config` and `/data/Config` are distinct paths; case-insensitivity allows confinement bypass.
  - **Remediation:** Implemented platform-aware component comparison (case-sensitive on Unix/Linux/macOS, case-insensitive only on Windows).
  - **Status:** Complete (Verified in `tests/security_test.nox`)

- [x] **SEC-02: Dotenv Root Confinement Conformance**
  - **File:** `env/source.nox`, `core/builder.nox`
  - **Issue:** `DotenvSource` reads paths directly via `read_to_string()` without enforcing `allowed_root` confinement.
  - **Remediation:** Added `allowed_root` parameter to `DotenvSource`, utilized `read_bounded_file()` from `sources.files`, and wired `with_allowed_root()` from `ConfigBuilder`.
  - **Status:** Complete (Verified in `tests/security_test.nox`)

- [x] **SEC-03: Real `nox.http` Transport for Remote HTTPS Provider**
  - **File:** `providers/remote_http.nox`
  - **Issue:** `HttpTransport.send_request()` returns mock 500; only `MockHttpTransport` exists.
  - **Remediation:** Implemented `StandardHttpTransport` using `nox.http.request` with SSL verification, timeout handling, and redirect control. Default to `StandardHttpTransport` when no mock is passed.
  - **Status:** Complete (Verified in `providers/remote_http.nox`, `tests/remote_provider_test.nox`)

- [x] **SEC-04: URL Allowlist SSRF Parser & Userinfo (@) Rejection**
  - **File:** `providers/remote_http.nox`
  - **Issue:** `_extract_host_port()` uses simple substring and prefix check `starts_with(allowed + ":")`, allowing userinfo SSRF bypass (e.g. `https://allowed.com:pass@evil.com/`).
  - **Remediation:** Implemented strict URL authority parser that explicitly rejects URLs containing `@` (userinfo credentials in authority) and validates host and port boundaries strictly.
  - **Status:** Complete (Verified in `tests/remote_provider_test.nox`)

---

## 🔒 P1: Snapshot Immutability & Secret API Contracts

- [x] **IMMUT-01: Snapshot Defensive Copying & Mutation Protection**
  - **File:** `core/snapshot.nox`, `core/types.nox`
  - **Issue:** `ConfigSnapshot` directly assigns the input dictionary `self.entries = entries`. Callers can mutate entries or values after publication.
  - **Remediation:** Deep-copy entries and `ConfigEntry` objects in `ConfigSnapshot.__init__` using `clone_config_entry()`. Published snapshot cannot be corrupted from outside.
  - **Status:** Complete (Verified in `tests/merge_test.nox`, `core/snapshot.nox`)

- [x] **SEC-05: Strict Secret Retrieval API Separation**
  - **File:** `core/snapshot.nox`
  - **Issue:** Calling `get_string()` on a key classified as a secret returns the plaintext secret without explicit intent, risking accidental logging.
  - **Remediation:** In `get_string()`, `get_int()`, `get_float()`, `get_bool()`, and `get_list()`, check `e.is_secret`. If true, raise `ConfigError(ERR_CONFIG_SECRET_SOURCE_DENIED, key, "key is classified as a secret: call get_secret() or get_secret_value() for explicit intentional access")`.
  - **Status:** Complete (Verified in `tests/integration_test.nox`, `tests/profiles_test.nox`)

- [x] **ENC-01: UTF-8 Codepoint vs Byte Indexing in .env Parser**
  - **File:** `env/parser.nox`
  - **Issue:** Quoted values loop using `byte_at()` up to `len(val_trimmed)` (which counts codepoints). Multibyte characters cause premature termination ("unclosed quote") and `char_from_byte()` produces corrupted mojibake.
  - **Remediation:** Refactored string scanning to iterate by Unicode code points `s[i]`. Added regression tests with Turkish and multibyte UTF-8 characters (`İstanbul`, `Türkçe`, `şemsiye`, `öğrenci`, `çözüm`).
  - **Status:** Complete (Verified in `tests/env_parser_test.nox`)

---

## ⚙️ P1: Configuration Lifecycle & Correctness

- [x] **ORD-01: Schema Defaults Prior to Interpolation**
  - **File:** `core/builder.nox`, `schema/validate.nox`
  - **Issue:** `build()` runs interpolation *before* schema validation adds default values. `${DEFAULT_KEY}` references cannot resolve schema defaults.
  - **Remediation:** Populated schema defaults into candidate entries *before* `resolve_interpolations()`.
  - **Status:** Complete (Verified in `tests/interpolation_test.nox`)

- [x] **VAL-01: Validation of Injected Schema Default Values**
  - **File:** `schema/validate.nox`
  - **Issue:** Default values are added to missing keys without running type, length, and range constraints on the default values themselves.
  - **Remediation:** Run full constraint validation on default values during schema initialization/application.
  - **Status:** Complete (Verified in `tests/schema_test.nox`)

- [x] **ROB-01: Single-Pass Forward Scanning in Redaction**
  - **File:** `secrets/redact.nox`
  - **Issue:** If a sentinel matches part of `[REDACTED]`, repeated `index_of()` causes an infinite loop.
  - **Remediation:** Implemented single-pass forward scanning that strictly advances past matches on remaining unredacted text.
  - **Status:** Complete (Verified in `tests/redaction_test.nox`)

- [x] **ROB-02: RFC 8259 Compliant JSON String Escaping**
  - **File:** `secrets/redact.nox`
  - **Issue:** Handwritten `escape_json_str()` does not escape all ASCII control characters (`\u0000`–`\u001f`).
  - **Remediation:** Leveraged `nox.json.dump_string(s)` for complete RFC 8259 escaping of all control characters and strings.
  - **Status:** Complete (Verified in `tests/redaction_test.nox`)

---

## 🚀 Release Engineering & CI Matrisi

- [x] **REL-01: Fix Release Pipeline Archive Generation**
  - **File:** `.github/workflows/release.yml`
  - **Issue:** `tar: .: file changed as we read it` occurred because archive was written inside the root directory being read.
  - **Remediation:** Used `git archive --format=tar.gz --output="$RUNNER_TEMP/vault-${TAG}.tar.gz"` and write distribution archives to `${{ runner.temp }}`.
  - **Status:** Complete

- [x] **CI-01: Dual Backend Verification**
  - **File:** `.github/workflows/ci.yml`, `.github/workflows/release.yml`
  - **Issue:** Ensure LLVM and QBE backends are explicitly tested in CI and documented.
  - **Remediation:** Added explicit execution steps for both `noxc test --backend qbe` and `noxc test --backend llvm` on all 4 platform runners.
  - **Status:** Complete

- [x] **VER-01: Bump Version to v1.0.1**
  - **Files:** `nox.json`, `vault.nox`, `CHANGELOG.md`, `README.md`, `cli/main.nox`
  - **Remediation:** Updated version to 1.0.1 across all documentation, metadata, test suites, and release notes.
  - **Status:** Complete
