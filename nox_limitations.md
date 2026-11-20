# NOX Compiler, Stdlib & Tooling Limitations Observed in VAULT

> **Current Validated Toolchain:** `noxc 2.2.38` (`7f879fe0ecb972dc9994ee1e8f420d4aaa0cdd9a`, `zig 0.16.0`, `x86_64-windows`, `ReleaseFast`)  
> **Previous Baseline:** `noxc 2.0.123` → `noxc 2.1.186`  
> **Last Updated:** 2026-11-20

This document tracks NOX compiler, standard-library, runtime, and CLI limitations encountered during **VAULT** development, clearly separating **items resolved in NOX `2.2.38`** from **active constraints still present in `2.2.38`**.

---

## 1. Active Limitations Remaining in `noxc 2.2.38`

### 1.1 `Exception` Subclasses Must Call `super().__init__(msg)` and Must Not Redeclare Inherited Fields
- **Observed Bug (`core/errors.nox`, `env/parser.nox`):** `core.nox` defines `Exception` with built-in fields `self.message: str`, `self.line: int`, and (`v2.2.22+`) `self.cause: str`. Declaring a subclass like `class ConfigError(Exception): message: str` *without* `super().__init__(msg)` caused:
  - Under `2.1.186`: `err.to_string()` read an uninitialized base `Exception.message`, printing `None`.
  - Under `2.2.38`: slot offset mismatch / uninitialized ARC pointer read `err.message` crashed with Windows **`0xC0000005` (`EXIT=5`)** on `cli_test.nox`.
- **Mandatory Convention:** Every `Exception` subclass MUST invoke `super().__init__(msg)` first and MUST NOT redeclare `message`, `line`, or `cause` (use distinct names such as `env_line` for `.env` line numbers since `raise` overwrites `self.line`).

### 1.2 `__all__` Rejects Module Constants & Re-Exports (`noxc 2.2.38`)
- **Observed (`vault.nox`, `PROBE-16`):** Declaring `__all__: list[str] = ["VERSION", ...]` in `vault.nox` fails compilation with:
  - `type error (UndefinedFunction): __all__ of module 'vault' lists 'VERSION', which is not a function or class defined there`.
  - Conversely, omitting a constant like `VERSION` from `__all__` causes external imports `from vault import VERSION` to fail with `'VERSION' is not part of the public API of module 'vault'`.
- **Workaround:** Enforce `__all__` on leaf function/class modules (`core/merge.nox`, `core/builder.nox`, `reload/bus.nox`) and omit `__all__` on constant/facade modules (`vault.nox`, `core/errors.nox`, `cli/main.nox`).

### 1.3 `nox.json.pos_int` Collapses Float Literals (`1.0` → `1`)
- **Observed (`sources/json.nox`):** Probing `{"flt": 1.0}` with `nox.json.pos_int(jv, p)` returned `int(1)` instead of distinguishing `1.0` as `float`.
- **Workaround:** Traverse containers and non-number scalars with zero-allocation `nox.json.pos_*`, and for `pos_kind == 2` delegate to `pos_value` + `nox.json.number_text()` so `"1.0"` stays `VAL_FLOAT` while `9007199254740993` stays `VAL_INTEGER`.

### 1.4 List Alias Growth Stale Reference (Pre-`2.2.43` Baseline)
- **Observation (`CHANGELOG.md [2.2.39]–[2.2.43]`):** In `2.2.38`, growing a list beyond capacity reallocates the buffer header so an alias `ys = xs` may not see later `xs.append(...)` (fixed on `main` in `2.2.43`, unreleased at `v2.2.38` pin).
- **Convention:** Always mutate/append on the canonical list owner field/variable rather than aliased locals; construct polymorphic `list[Base]` containers via `xs: list[Base] = []` + `xs.append(...)` (`PROBE-02`).

### 1.5 Optional (`T | None`) Narrowing Rules
- **Observation (`PROBE-03`):** Scalar `int | None` fields or non-narrowed comparisons outside `if x is not None:` / `and` / pointer-attr checks require assigning to a local or explicit `is not None` guard before field/method access.

### 1.6 Multi-Line Parenthesized Imports & `noxclibs` Portability
- **Note:** Keep `sources/json5.nox` (`nox.yyjson`), `schema/pcre2.nox` (`nox.pcre2` requiring `dynamic_code`), and `sources/fast_hash.nox` (`nox.xxhash`) out of `vault.nox` so `import vault` links zero `noxclibs.a` C libraries (`nox:native: "false"`). Also preserve Windows case-insensitive Win32 `_putenv_s` vs POSIX case-sensitive env semantics (`tests/env_process_test.nox` & `sources/files.nox`).

### 1.7 `nox.process.Command` POSIX Bare-Name `PATH` Lookup (`ProcessError`)
- **Observed (`tests/cli_test.nox` on Linux x64, Linux ARM64, macOS ARM64):** `runtime/stdlib_shims/process.zig` (`v2.2.38`) passes `snapshotEnv` to `std.process.run`'s `.environ_map`, but Zig's `Threaded.spawnPosix` resolves bare `argv[0]` (`"noxc"`) via `t.environ.string.PATH` on the shared `http_client.sharedClientIo()` (`Environ.empty`), falling back to `default_PATH` (`/usr/local/bin:/bin:/usr/bin`). Binaries installed under `$HOME/.nox-lang/bin` therefore fail with `ProcessError` on POSIX even when `PATH` contains them (Windows `CreateProcessW` searches `PATH` and passed).
- **Workaround:** Resolve executable paths (`resolve_noxc()` in `tests/cli_test.nox` via `$HOME/.nox-lang/bin/noxc` / `PATH` + `nox.fs.is_file`) and pass the explicit path to `Command(...)`.

---

## 2. Historical NOX `2.1.186` Limitations Resolved by NOX `2.2.38`

| Former `2.1.186` Limitation | Upstream NOX Resolution | Integrated in VAULT `v1.2.0` |
| :--- | :--- | :--- |
| No HTTP socket pinning / SNI split | `nox.http.request(..., connect_ip=, ca_file=)` (`v2.1.195–196`), `nox.tls.connect(..., connect_ip=)` (`v2.2.1`), `nox.safehttp` (`v2.2.1`, `v2.2.25`) | **`PinnedHttpTransport`** (`providers/remote_http.nox`) + RFC 6454 origin redirect stripping (`NET-PIN-UPSTREAM` unblocked!) |
| No built-in DNS resolver (`nslookup` subprocess) | `nox.dns.resolve(host, timeout_ms, doh_url)` (`v2.1.195`, `v2.2.4`) + `nox.ip` (`v2.2.1`) | **`SystemDnsResolver`** uses native `nox.dns.resolve` + `nox.ip.is_global_text` |
| No symmetric encryption in `nox.crypto` | `aead_seal`/`aead_open` + `*_bytes` (XChaCha20-Poly1305, ChaCha20-Poly1305, AES-GCM, `v2.1.195–2.2.1`) | **`secrets/sealed.nox`**, **`sources/sealed.nox`**, `ConfigBuilder.with_sealed_file()` |
| No OS env enumeration (`nox.os.environ()`) | `nox.os.environ()` & `env_names()` (`v2.1.195`) | **`ProcessEnvSource.from_prefix()`** & `ConfigBuilder.with_env_prefix()` |
| Nested `noxc run` Windows `FileNotFound` (`Command`) | Parent env snapshot in `nox.process.Command` (`v2.2.0-rc.2`) | Restored live subprocess CLI isolation checks in `tests/cli_test.nox` |
| `noxc publish --help` submitted `--help` as `<repo>` | Fixed CLI arg handling (`v2.1.195`) | Verified in `2.2.38` tooling notes |
| `noxc sbom` emitted `nox:native="true"` for `"bin"`-only pure-NOX pkgs | Fixed (`v2.1.195`) | Verified locally on `2.2.38`: `"nox:native": "false", "nox:bin": "true"` |
