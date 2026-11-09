# VAULT Security Model & Threat Assessment

This document outlines the security architecture, threat mitigations, and formal guarantees enforced by the **VAULT** configuration engine.

---

## 1. Core Threat Mitigations

| Threat | Attack Vector | VAULT Mitigation | Formal Reference |
| :--- | :--- | :--- | :--- |
| **Secret Leakage in Logs** | Accidental printing or JSON serialization of database credentials | Structured automatic redaction: secrets serialize as `"[REDACTED]"`. | `SECRET-01`, `SECRET-02` |
| **Transitive Secret Exposure** | Secrets concatenated or interpolated into public URLs | Graph-based taint propagation: any setting referencing a secret becomes secret. | `INTERP-04` |
| **Directory Traversal** | Malicious paths (e.g. `../../etc/passwd`) | Path-segment-aware root confinement: canonicalizes and rejects escapes. | `FILE-01`, `FILE-02` |
| **Symlink Escapes** | Symlinks pointing outside permitted root | Resolves symlinks before canonical segment check. | `FILE-03` |
| **SSRF (Server-Side Request Forgery)** | Secret provider targeting AWS/GCP metadata (`169.254.169.254`) | Rejects non-HTTPS schemes and link-local IP addresses before making requests. | `PROVIDER-01`, `PROVIDER-02` |
| **Credential Hijacking via Redirects** | Remote provider redirected to attacker server (HTTP 302) | Inspects `Location` header before sending credentials; blocks unapproved hops. | `PROVIDER-02` |
| **Denial-of-Service (Expansion Bomb)** | `${A}` -> `${B}${B}` exponential string expansion | Bounded depth (16 hops) and total expansion limit (128 KiB). | `INTERP-03` |
| **Resource Exhaustion** | Multi-gigabyte `.env` or config files | Hard 1 MiB file size limit and 2,000 key limits per source. | `FILE-04`, `FILE-05` |
| **Corrupt State Downtime** | Bad config update crashes live service | Atomic validate-before-swap: preserves last-known-good configuration on error. | `RELOAD-01` |
| **Host Environment Pollution** | `.env` parser mutating OS environment | Pure parser: loading `.env` never executes `nox.os.set_var`. | `ENV-01` |

---

## 2. Structural Redaction Architecture

VAULT enforces **structural AST-level redaction** as the primary security layer, supplemented by string-level sentinel pattern scanning:

1. **Tagging:** Every secret field declared in schema or fetched via `SecretProvider` is tagged with `is_secret = true`.
2. **Taint Propagation:** During `${KEY}` resolution, the dependency graph tracks secret ancestors. If any ancestor is secret, the descendant is automatically tainted.
3. **Safe Serialization (`snapshot.redacted_json()`):** During serialization, any key where `is_secret == true` outputs `"[REDACTED]"`.
4. **Non-Sensitive Provenance (`snapshot.explain(key)`):** Diagnostics output winning rank, source kind, and profile, but **never** the secret value.
5. **Defense-in-Depth (`redact_string`):** Secondary scanner scans log output against active sentinels.

---

## 3. Remote Secret Provider Isolation

When consuming remote HTTPS secret stores:
1. **Scheme Enforcement:** HTTPS is strictly mandatory (`https://`). Cleartext HTTP and FTP are rejected immediately.
2. **Metadata IP Blacklisting:** IP addresses beginning with `169.254.` (AWS, GCP, Azure, OpenStack instance metadata) are blocked regardless of allowlist entries.
3. **Pre-Flight Redirect Validation:** HTTP 301, 302, 307, 308 redirects are intercepted. If the destination URL is not an approved HTTPS host, the request terminates immediately **without sending authentication headers**.
4. **Token Non-Disclosure:** If the provider responds with 401 or 403, error diagnostics report `CONFIG_SECRET_SOURCE_DENIED` and strictly omit the bearer token.

---

## 4. Snapshot Immutability, Secret Gate & Encapsulation Boundary (IMMUT-04 / IMMUT-05)

1. **Defensive Property Encapsulation:**
   - `ConfigSnapshot.entries` is exposed strictly as a `@property` returning an isolated dictionary.
   - For every entry marked as secret (`is_secret == true`), the returned `ConfigEntry` has its underlying value strictly replaced with `make_string("[REDACTED]")`.
   - Modifying or mutating the dictionary or `ConfigEntry` objects returned by `.entries` has zero effect on the internal snapshot state.
2. **Strict Secret Gate:**
   - Normal getters (`get_string()`, `get_int()`, `get_bool()`, `get_entry()`) reject secrets fail-closed with `ERR_CONFIG_SECRET_SOURCE_DENIED`.
   - Explicit retrieval requires deliberate caller intent via `get_secret(key)` or `get_secret_entry(key)`.
3. **Encapsulation Boundary & Language Constraints (IMMUT-05):**
   - Internal raw configuration records reside in `self._entries`.
   - In accordance with NOX 2.x language conventions, symbols with a leading underscore `_` are strictly private library implementation details. Application code MUST NOT bypass public accessors to inspect private fields.

---

## 5. SSRF Defense, Pre-Flight DNS Resolution & DNS Pinning (NET-03 / NET-04)

1. **Mathematical IPv4 CIDR Validation:**
   - Host IP addresses are converted into 32-bit unsigned integers and validated against exact binary bitmasks.
   - Prohibits: Loopback (`127.0.0.0/8`), RFC 1918 Private (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), Cloud Link-Local Metadata (`169.254.0.0/16`), Carrier-Grade NAT (`100.64.0.0/10`), Multicast (`224.0.0.0/4`), Unroutable (`0.0.0.0/8`), Broadcast (`255.255.255.255`), and Reserved (`240.0.0.0/4`).
2. **IPv6 Subnet Validation:**
   - Validates Loopback (`::1`), Link-Local (`fe80::/10`), Unique Local ULA (`fc00::/7`), IPv4-mapped (`::ffff:0:0/96`), and Multicast (`ff00::/8`).
3. **Numeric Host Obfuscation Rejection:**
   - Hostnames containing DWORD decimal integers (`2130706433`), octal notation (`0177.0.0.1`), abbreviated formats (`127.1`), or hexadecimal (`0x7f000001`) are rejected outright.
4. **Pre-Flight DNS Resolution & Rebinding Defense:**
   - `RemoteHttpSecretProvider` resolves hostnames via `DnsResolver` prior to issuing network requests.
   - If any resolved IP belongs to a prohibited or private range, the provider fails closed with `ERR_CONFIG_PROVIDER_UNAVAILABLE`.
5. **DNS Pinning (TOCTOU Elimination):**
   - The connection target is pinned directly to the verified IP address (`https://[pinned_ip]/path`), transmitting the original domain name in the HTTP `Host:` header. This eliminates Time-Of-Check to Time-Of-Use (TOCTOU) DNS rebinding attacks.

---

## 6. Atomic Concurrency & Unified Publication (RELOAD-03 / RELOAD-04)

1. **Hardware-Level Atomic CAS Coordination:**
   - `ReloadManager` reload execution is guarded by `nox.atomic.new_bool(False)` utilizing `compare_and_swap(False, True)`.
   - Eliminates thread/worker race conditions during concurrent reloads.
2. **Unified Atomic Publication Record (`PublishedConfig`):**
   - Version number, immutable snapshot, timestamp, and metrics are encapsulated within a single immutable `PublishedConfig` object.
   - Updates are published via an atomic single-pointer swap (`self._published = new_published`). Readers calling `manager.get_published()` or `manager.snapshot()` observe a 100% synchronized, non-torn state.
3. **Cooperative Fiber Scheduling:**
   - Async reload routines yield cooperatively via `spawn` and `await`, ensuring genuine task interleaving across both single-threaded cooperative (QBE) and parallel multi-core (LLVM) execution environments.
