# NOX v2.1.186 Compatibility & Capability Probe Report

> **Compiler Baseline:** `noxc 2.1.186` (upgraded from `2.0.123` / `2.1.0`)  
> **Platform Tested:** Windows 10/11 x64 (cross-checked in CI on Linux x64, Linux ARM64, macOS ARM64, Windows x64)  
> **Backends Tested:** LLVM (`noxc test --backend llvm`), QBE (`noxc test --backend qbe`)  
> **Profile:** Hosted (`vault.nox` core remains portable and free of `noxclibs` / `dynamic_code`)  
> **Date:** 2026-11-16

---

## 1. Capability Probes Summary

| Probe ID | Module / Feature | Status | Key Discoveries & Platform Facts |
| :--- | :--- | :--- | :--- |
| **PROBE-01** | Custom Exceptions & Error Codes | **PASS** (LLVM & QBE) | Custom exception classes with `code`, `key`, `message` work cleanly across backends. |
| **PROBE-02** | Collections & O(1) Tombstone Deletion | **PASS** | `dict.pop(key)` (v2.1.3+) and `Set.remove` (v2.1.186) execute in amortized O(1) with tombstones. Polymorphic container population must use explicit `.append()` on base-typed local `list[Base]`. |
| **PROBE-03** | Nullability & Value States | **PASS** | `str \| None` narrowing is reliable inside explicit `if x is not None:` blocks. Avoid equality comparisons on un-narrowed `T \| None`. |
| **PROBE-04** | Class Polymorphism & Injection | **PASS** | Virtual dispatch (`ConfigSource`, `SecretProvider`, `HttpTransport`, `DnsResolver`) behaves identically on LLVM and QBE. |
| **PROBE-05** | OS Environment & Missing Behavior | **PASS** | `nox.os.getenv(name)` raises `OsError` when absent. Catch requires explicit `from nox.os import OsError`. |
| **PROBE-06** | File Metadata & `nox.path.safe_join` | **PASS** | `nox.fs.read_to_string` reads directly into target `str` buffer (v2.1.169). `nox.path.safe_join` + `strip_prefix` operate cleanly on segment boundaries; VAULT pairs them with OS case-awareness on Windows. |
| **PROBE-07** | Exact JSON Int64 & Lazy Tape Parser | **PASS** | `nox.json.parse` uses single-pass lazy tape parser (v2.1.167-168); `nox.json.as_int` & `number_text` preserve exact 64-bit integers (`9007199254740993`). `JsonWriter` + `nox.strings.join` avoid O(n²) concatenation. |
| **PROBE-08** | Strings, BOM, CRLF & Inline `len(str)` | **PASS** | Native string slicing handles UTF-8 codepoints; `len(str)` uses inline ASCII fast-path (v2.1.163). |
| **PROBE-09** | Backend Parity (QBE vs LLVM) | **PASS** (Both) | M:N work-stealing & atomic ARC now span both LLVM and QBE (v2.1.112+); `nox.thread.start` takes exactly 2 args `(entry, arg)`. |
| **PROBE-10** | `AtomicInt`, `AtomicBool`, `AtomicRef` | **PASS** | `nox.atomic.AtomicRef` (`new_ref`, `from_ref_handle`, `get`, `set`, `version` starts at `0`, `free`) shares untorn byte snapshots across isolated OS worker heaps. |
| **PROBE-11** | `nox.crypto` (`blake3`) & `nox.ed25519` | **PASS** | `nox.crypto.blake3`, `sha3_256`, `constant_time_eq`, and `nox.ed25519.verify` / `fingerprint` pass RFC 8032 vectors with fail-closed behavior on malformed hex. |
| **PROBE-12** | `nox.argparse` & `nox.property` | **PASS** | Pure-NOX `ArgumentParser` handles subcommands, `--opt=val`, flags, and positionals. `nox.property.check` runs seeded deterministic fuzzing with shrinking. |
| **PROBE-13** | Opt-In `noxclibs` (`yyjson`, `pcre2`, `xxhash`) | **PASS** (Hosted) | Embedded C archive `noxclibs.a` is linked **only** when opt-in submodules (`sources.json5`, `schema.pcre2`, `sources.fast_hash`) are imported. Core `import vault` does not pull `noxclibs` or require `dynamic_code`. |
| **PROBE-14** | Monotonic Time (`monotonic_ns`, `boot_ms`) | **PASS** | `nox.time.monotonic_ns()` provides nanosecond timing; `nox.time.sleep_ms()` yields cooperatively inside async fibers on macOS, Linux, and Windows (`WSAPoll`). |
| **PROBE-15** | Module Import Isolation & HTTP Pinning Boundary | **PASS / BLOCKED (Upstream)** | Moving CLI execution out of `cli/main.nox` to `cli/entry.nox` ensures `import vault` has zero CLI/process side effects. Note: `nox.http.request` and `nox.tls.connect` in NOX `2.1.186` still do not accept separate `connect_ip` + SNI socket pinning; `StrictPinnedHttpTransport` remains fail-closed. |

---

## 2. Core vs Opt-In C-Library Capability Matrix

| Module | Underlying NOX Stdlib | Included in `import vault`? | Supported Targets | Required Extra Capabilities |
| :--- | :--- | :---: | :--- | :--- |
| `vault` (Core, CLI, Signed, Bus) | `nox.os`, `nox.fs`, `nox.path`, `nox.json`, `nox.toml`, `nox.yaml`, `nox.crypto`, `nox.ed25519`, `nox.atomic`, `nox.regex`, `nox.url`, `nox.argparse` | Yes | Hosted desktop, WASM, Mobile, Freestanding (pure subset) | None beyond standard hosted `filesystem`/`clock`/`network` when used |
| `sources.json5` | `nox.yyjson` | **Opt-In Only** | Hosted (macOS, Linux, Windows) | Links `noxclibs.a` |
| `schema.pcre2` | `nox.pcre2` | **Opt-In Only** | Hosted (macOS, Linux, Windows) | Links `noxclibs.a` + `dynamic_code` (JIT) |
| `sources.fast_hash` | `nox.xxhash` | **Opt-In Only** | Hosted (macOS, Linux, Windows) | Links `noxclibs.a` |
