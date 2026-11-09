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
