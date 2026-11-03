# NOX v2.0.123 Compatibility & Capability Probe Report

> **Compiler Baseline:** `noxc 2.0.123`  
> **Platform Tested:** Windows 10/11 x64 (PowerShell environment)  
> **Backends Tested:** LLVM (default), QBE (`--backend qbe`)  
> **Profile:** Hosted  
> **Date:** 2026-11-03  

---

## 1. Capability Probes Summary

| Probe ID | Module / Feature | File Path | Status | Key Discoveries & Platform Facts |
| :--- | :--- | :--- | :--- | :--- |
| **PROBE-01** | Custom Exceptions & Error Codes | `tests/probes/probe_errors.nox` | **PASS** (LLVM & QBE) | Custom exception classes with fields (`code: str`, `key: str`) work cleanly. Top-level statements are the entry point; `main` is a reserved name. |
| **PROBE-02** | Collections (`dict`, `list`, objects) | `tests/probes/probe_collections.nox` | **PASS** | `dict[str, str] = {}` and `list[T] = []` work with type annotations. `.append()` receiver must be a bare local variable name. |
| **PROBE-03** | Nullability & Value States | `tests/probes/probe_nullability.nox` | **PASS** | `str | None` supported. Comparing `str | None` with `str` is a compile error (`TypeMismatch`). `if x is not None:` performs type narrowing to `str`. |
| **PROBE-04** | Class Polymorphism & Injection | `tests/probes/probe_callbacks.nox` | **PASS** | Class inheritance and dynamic dispatch (`BaseProvider` -> `MemoryProvider`) work cleanly. Verified pattern for `ConfigSource` and `SecretProvider`. |
| **PROBE-05** | OS Environment & Missing Behavior | `tests/probes/probe_os.nox` | **PASS** | `nox.os.getenv(name)` raises `OsError` when absent. Catching requires simple identifier (`from nox.os import OsError`), not dotted name. |
| **PROBE-06** | File Metadata & Canonicalization | `tests/probes/probe_fs.nox` | **PASS** | `nox.fs.metadata(path).size` accurately reports file size before buffering. `nox.path.canonicalize` and `nox.path.components` work for sandboxed path-segment checks. |
| **PROBE-07** | Structured Formats (JSON/TOML/YAML)| `tests/probes/probe_formats.nox` | **PASS** | `JsonValue` is globally available from core. `TomlValue` and `YamlValue` require explicit import from `nox.toml` and `nox.yaml`. All three parse nested structures. |
| **PROBE-08** | Strings, BOM, CRLF & Slicing | `tests/probes/probe_strings.nox` | **PASS** | Native string slicing `s[0:5]` works. Lexer does not parse literal `\r`; carriage return must be handled via `nox.strings.char_from_byte(13)` or `byte_at == 13`. |
| **PROBE-09** | Backend Parity (QBE vs LLVM) | `tests/probes/probe_errors.nox` | **PASS** (Both) | Error handling and semantics match identically across both LLVM and QBE backends. |
| **PROBE-10** | Atomic Primitives | `nox.atomic` inspection | **PASS** | `AtomicInt` and `AtomicBool` are available in `nox.atomic` for thread synchronization if needed. |

---

## 2. Standard Library Verified Signatures

### `nox.os`
- `getenv(name: str) -> str`: Raises `OsError` if missing (never returns empty string sentinel).
- `set_var(name: str, value: str) -> None`: Sets environment variable for process and children.
- `current_dir() -> str`: Returns current working directory.
- `exit(code: int) -> None`: Terminates current process with status code.

### `nox.fs`
- `metadata(path: str) -> FileMetadata`: Provides `.size: int` and `.modified_ms: int`.
- `read_to_string(path: str) -> str`: Reads entire file; raises `FsError` on failure.
- `write_string(path: str, content: str) -> None`: Writes file content.
- `exists(path: str) -> bool`: Checks existence without raising.
- `is_file(path: str) -> bool`: Checks whether path is a regular file.
- `remove_file(path: str) -> None`: Deletes file.

### `nox.path`
- `canonicalize(p: str) -> str`: Resolves symlinks and normalizes; raises `PathError` if unresolved.
- `components(p: str) -> list[str]`: Splits path into components for safe path-segment prefix verification.
- `join(a: str, b: str) -> str`: Safe path joining.
- `is_absolute(p: str) -> bool`: Checks whether path is absolute.

### `nox.strings`
- `byte_at(s: str, idx: int) -> int`: Safe zero-copy byte inspection.
- `char_from_byte(b: int) -> str`: Convert byte value (0-255) to 1-char string.
- `byte_len(s: str) -> int`: Length in bytes.
- `split(s: str, sep: str) -> list[str]`: Splits string by separator.
- `splitn(s: str, sep: str, n: int) -> list[str]`: Splits into at most `n` parts.
- `trim(s: str) -> str`, `trim_start(s: str) -> str`, `trim_end(s: str) -> str`: Whitespace trimming.
- `replace(s: str, old: str, new: str) -> str`: Replaces occurrences.
- `starts_with(s: str, prefix: str) -> bool`, `ends_with(s: str, suffix: str) -> bool`.
- `index_of(s: str, needle: str) -> int`, `contains(s: str, needle: str) -> bool`.
- `eq_ignore_case(a: str, b: str) -> bool`: Case-insensitive comparison.

### `nox.json`
- `parse(s: str) -> JsonValue`: Raises `JsonError` on invalid JSON.
- `is_null`, `is_bool`, `is_number`, `is_string`, `is_array`, `is_object`.
- `as_bool(v: JsonValue) -> bool`, `as_number(v: JsonValue) -> float`, `as_string(v: JsonValue) -> str`.
- `array_len(v: JsonValue) -> int`, `array_get(v: JsonValue, i: int) -> JsonValue`.
- `object_len(v: JsonValue) -> int`, `object_key(v: JsonValue, i: int) -> str`, `object_value(v: JsonValue, i: int) -> JsonValue`.

### `nox.toml`
- `parse(s: str) -> TomlValue`: Raises `TomlError` on invalid syntax or duplicate keys.
- `TomlValue` properties: `kind: int` (0=string, 1=int, 2=float, 3=bool, 4=array, 5=table, 6=datetime), `tbl: dict[str, TomlValue]`, `arr: list[TomlValue]`.
- Helpers: `is_string`, `is_int`, `is_float`, `is_bool`, `is_array`, `is_table`.

### `nox.yaml`
- `parse(s: str) -> YamlValue`: Raises `YamlError` on invalid syntax.
- `YamlValue` properties: `kind: int` (0=string, 1=int, 2=float, 3=bool, 4=null, 5=sequence, 6=mapping), `mapping: dict[str, YamlValue]`, `arr: list[YamlValue]`.
- Helpers: `is_string`, `is_int`, `is_float`, `is_bool`, `is_null`, `is_sequence`, `is_mapping`.
