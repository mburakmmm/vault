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

## 5. SSRF Defense, Pre-Flight DNS Resolution & TLS-Aware Pinning (NET-04 / NET-05 / NET-06)

1. **Exact 128-Bit IPv6 Parser & Strict Mathematical CIDR Evaluation (NET-06):**
   - Full RFC 4291 parsing produces 8 exact 16-bit hextet integers (`list[int]`), supporting standard compression (`::`), hex conversion, and embedded IPv4 suffixes (`::ffff:a.b.c.d`).
   - Bitwise mask evaluations mathematically verify and reject:
     - Unspecified (`::/128`) and Loopback (`::1/128`)
     - Link-Local Unicast (`fe80::/10` -> `(h0 & 65472) == 65152`)
     - Unique Local Address ULA / Private (`fc00::/7` -> `(h0 & 65024) == 64512`)
     - Multicast (`ff00::/8` -> `(h0 & 65280) == 65280`)
     - Discard Prefix (`100::/64`) and Documentation (`2001:db8::/32`, `2001:2::/48`)
     - IPv4-Mapped (`::ffff:0:0/96`), IPv4-Compatible (`::0:0/96`), 6to4 (`2002::/16`), and Teredo (`2001:0000::/32`) with embedded 32-bit IPv4 extraction and binary CIDR evaluation.
   - **Fail-Closed Malformed Input Policy:** Malformed IP representations (e.g. `256.1.1.1`, `fe80:::1`, `1.2.3.4.5`) fail closed immediately; they are never loosely treated as valid hostnames.

2. **Mathematical 32-Bit IPv4 CIDR Validation:**
   - Host IP addresses are converted into 32-bit integers and validated against exact binary bitmasks.
   - Prohibits: Loopback (`127.0.0.0/8`), RFC 1918 Private (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), Cloud Link-Local Metadata (`169.254.0.0/16`), Carrier-Grade NAT (`100.64.0.0/10`), Multicast (`224.0.0.0/4`), Unroutable (`0.0.0.0/8`), Broadcast (`255.255.255.255`), and Reserved (`240.0.0.0/4`).

3. **Numeric Host Obfuscation Rejection:**
   - Hostnames containing DWORD decimal integers (`2130706433`), octal notation (`0177.0.0.1`), abbreviated formats (`127.1`), or hexadecimal (`0x7f000001`) are rejected outright.

4. **Pre-Flight DNS Resolution & TOCTOU Defense:**
   - `RemoteHttpSecretProvider` resolves hostnames via `DnsResolver` prior to issuing network requests.
   - If any resolved IP belongs to a prohibited or private range, the provider fails closed with `ERR_CONFIG_PROVIDER_UNAVAILABLE` before touching the network transport.

5. **TLS-Aware Pinning & Upstream Standard Library Protocol (NET-05 / NET-07):**
   - In accordance with TLS RFC specifications, replacing a URL host with an IP address breaks TLS certificate validation when certificates lack IP Subject Alternative Names (SANs).
   - VAULT maintains the authentic HTTPS URL (`https://<hostname>/<path>`) and transmits `Host: <hostname>`, while explicitly passing `connect_ip` and `tls_server_name` to `HttpTransport.send_request`.
   - **Socket-Pinning Fail-Closed Enforcement (NET-07):** When strict socket pinning is configured (`enforce_socket_pinning=True`) and `connect_ip` differs from `tls_server_name`, `StandardHttpTransport` fails closed immediately with `ERR_CONFIG_PROVIDER_UNAVAILABLE` rather than silently issuing an unpinned request.
   - **Authentic System DNS Resolution (NET-07):** `SystemDnsResolver` executes OS-level resolution (`nslookup`) to resolve genuine A/AAAA DNS records before pre-flight SSRF and CIDR evaluation.
   - **RFC 4291 Strict IPv6 Grammar (NET-08):** Strictly rejects triple colons (`:::`, `fe80:::1`, `2001:4860:::8888`), single leading/trailing colons (`:2001::1`, `2001::1:`), multiple `::` abbreviations, empty blocks, and invalid hex ranges.

---

## 6. Atomic Concurrency & Unified Immutable Publication (RELOAD-04 / RELOAD-05 / RELOAD-06)

1. **Hardware-Level Atomic CAS Coordination & Race Elimination (RELOAD-06):**
   - `ReloadManager` reload execution is guarded by `nox.atomic.new_bool(False)` utilizing `compare_and_swap(False, True)`.
   - `current_pub: PublishedConfig = self._published` is read strictly *after* acquiring the CAS lock, eliminating the race window where concurrent writers could read a stale snapshot and compute duplicate version numbers.
   - Eliminates thread/worker race conditions during concurrent reloads. Multi-writer collisions safely return `CONFIG_RELOAD_IN_PROGRESS`.

2. **Truly Immutable Publication Record (`PublishedConfig`):**
   - Version number, immutable snapshot, epoch timestamp, and reload count are encapsulated in `PublishedConfig`.
   - All internal fields (`_version`, `_snapshot`, `_published_at_ms`, `_reload_count`) are private, exposed via read-only `@property` getters.
   - `published_at_ms` is initialized with genuine `nox.time.now_ms()` clock telemetry upon creation and publication.
   - State swaps execute via an atomic pointer assignment (`self._published = new_published`) immediately followed by an atomic store release barrier (`self._atomic_reloading.store(False)`), guaranteeing cross-thread and cross-worker memory visibility.

3. **Cooperative Fiber Scheduling:**
   - Async reload routines yield cooperatively via `spawn` and `await`, ensuring genuine task interleaving across both single-threaded cooperative (QBE) and parallel multi-core (LLVM) execution environments.
