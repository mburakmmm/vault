<div align="center">

# 🔒 VAULT
**Configuration, Environment & Secret Management for the NOX Ecosystem**

[![NOX](https://img.shields.io/badge/NOX-v2.0.123-blue.svg?style=flat-square)](https://github.com/mburakmmm/nox-lang)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat-square)](LICENSE)
[![Status: Phase 0 Complete](https://img.shields.io/badge/Status-Phase%200%20Verified-brightgreen.svg?style=flat-square)](todo.md)
[![Security: Fail--Closed](https://img.shields.io/badge/Security-Fail--Closed-red.svg?style=flat-square)](SECURITY.md)

*A reusable, framework-independent, type-safe configuration engine with deterministic precedence and zero secret leakage.*

</div>

---

## 📖 Overview

**VAULT** provides a rigorous, fail-closed configuration and secret management foundation for [NOX](https://github.com/mburakmmm/nox-lang) applications, command-line tools, and backend services.

It delivers a predictable, auditable answer to:
- **Where did this setting come from?** (*Traceable provenance without revealing secrets*)
- **Is it valid?** (*Aggregated, typed schema validation before network or database startup*)
- **Can it be inspected or logged without leaking secrets?** (*Guaranteed structural redaction*)
- **Can it be updated safely at runtime?** (*Validate-before-swap atomic reloads*)

---

## 🏛️ Architecture

```text
                         +---------------------------+
                         | Application / Service     |
                         | Aether / Nyx / CLI / etc. |
                         +-------------+-------------+
                                       |
                         +-------------v-------------+
                         |  vault.nox (Public Facade)|
                         +-------------+-------------+
                                       |
        +------------------------------+-------------------------------+
        |                              |                               |
  +-----v------+                +------v------+                 +------v------+
  | vault.core |<---------------| vault.schema|                 | vault.secrets|
  | Values     |                | Validate    |                 | Labels, Safe|
  | Snapshots  |                | Typed Views |                 | Redaction   |
  +-----+------+                +-------------+                 +-------------+
        |
  +-----v---------------------------------------------------------------+
  | vault.sources / vault.env / vault.interpolate / vault.profiles      |
  | Memory, Dotenv, Process Env, JSON, TOML, YAML, Profiles             |
  +--------------------------+------------------------------------------+
                             |
                    +--------v---------+
                    | Optional Providers|  Env, Root-Confined Files, HTTPS
                    +------------------+
```

---

## ✨ Core Pillars & Invariants

1. **NOX First & Standalone:**  
   Zero external runtime dependencies. Built exclusively using `.nox` modules and the NOX standard library (`nox.os`, `nox.fs`, `nox.path`, `nox.strings`, `nox.json`, `nox.toml`, `nox.yaml`, `nox.crypto`, `nox.log`, `nox.time`).
2. **Safe by Default:**  
   Parsing `.env` files **never** mutates the process environment (`nox.os.set_var`). It yields an isolated, immutable configuration snapshot.
3. **Fail-Closed Secrecy:**  
   Secret values are wrapped in dedicated types and automatically rendered as `[REDACTED]` in diagnostics, errors, JSON exports, and logs. Required secrets missing in production cause an immediate startup failure.
4. **Deterministic Precedence:**  
   Every configuration key resolves to exactly one winning source through a predictable precedence hierarchy with verifiable origin metadata.
5. **Pure & Declarative `.env`:**  
   `.env` files are pure data — never shell scripts. Absolutely no `eval`, command substitution (`$(...)`), backticks, or subprocess execution.
6. **Bounded Resource Usage:**  
   Hard ceilings prevent resource exhaustion attacks:
   - Max file size: **1 MiB**
   - Max keys per source: **2,000**
   - Max single value: **64 KiB**
   - Max key length: **128 bytes**
   - Max interpolation depth: **16**
   - Max expanded value: **128 KiB**

---

## ⚖️ Deterministic Source Precedence

Configuration sources are evaluated in a strictly ordered, documented sequence (lowest to highest priority):

| Rank | Source Slot | Description |
| :---: | :--- | :--- |
| **1** | Field / Schema Defaults | Hardcoded safe fallbacks declared in schema |
| **2** | In-Memory Defaults | Application-level base dictionary |
| **3** | Base Config File | `vault.toml`, `vault.yaml`, or `vault.json` |
| **4** | Profile File | `vault.production.toml`, `vault.development.yaml` |
| **5** | Base `.env` File | Standard project `.env` file |
| **6** | Profile `.env.<profile>` | Environment-specific override file (`.env.production`) |
| **7** | Secret Providers | Local or remote secret adapters (for declared secret keys) |
| **8** | Process Environment | Explicitly allowed / schema-mapped OS environment keys only |
| **9** | Runtime / Test Overrides | Explicit values passed directly into the builder |

> *A higher-priority value that fails validation immediately fails configuration loading — it never silently falls back to a lower-priority value.*

---

## 🚀 Quick Preview

```nox
import vault

# Version & facade check
print("VAULT Version: " + vault.version())
```

---

## 🗺️ Roadmap & Phase Acceptance

Detailed tasks and verification criteria are tracked in [todo.md](todo.md):

- [x] **v0.1.0 — Phase 0:** Capability Audit, NOX Probes & Package Skeleton
- [ ] **v0.2.0 — Phase 1:** Pure & Bounded `.env` Parser
- [ ] **v0.2.5 — Phase 2:** Core Snapshots, Sources & Deterministic Precedence
- [ ] **v0.3.0 — Phase 3:** Typed Schema & Aggregate Validation (**MVP Milestone**)
- [ ] **v0.4.0 — Phase 4:** Structured Sources (JSON, TOML, YAML Bridges) & Root-Constrained Files
- [ ] **v0.5.0 — Phase 5:** Profiles, Variable Interpolation & Secret Taint
- [ ] **v0.6.0 — Phase 6:** Secret Classification, Redaction & Safe Inspection (**Practical Package**)
- [ ] **v0.7.0 — Phase 7:** Local Secret Providers (Env, File, Memory)
- [ ] **v0.8.0 — Phase 8:** Atomic Reload & Last-Known-Good Snapshots (**Core Complete**)
- [ ] **v0.9.0 — Phase 9:** Optional Remote Providers (HTTPS & SSRF Defense)
- [ ] **v0.9.5 — Phase 10:** Optional CLI Tooling (`vault validate`, `explain`)
- [ ] **v0.9.8 — Phase 11:** Ecosystem Adapters (Aether, Nyx, Feather, etc.)
- [ ] **v1.0.0 — Phase 12:** Hardening, Full Regression Matrix & Stable Release

---

## 🧪 Testing

VAULT enforces real test execution with zero mock assertions:

```bash
# Static type checking
noxc check vault.nox

# Run test suite
noxc test
```

All test runs require passing assertions and clean diagnostic scans.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
