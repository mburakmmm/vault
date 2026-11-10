<div align="center">

# 🔒 VAULT
**Configuration, Environment & Secret Management for the NOX Ecosystem**

[![CI](https://github.com/mburakmmm/vault/actions/workflows/ci.yml/badge.svg)](https://github.com/mburakmmm/vault/actions/workflows/ci.yml)
[![Platforms](https://img.shields.io/badge/Platforms-macOS%20ARM%20%7C%20Linux%20x64%20%7C%20Linux%20ARM%20%7C%20Windows%20x64-blue.svg?style=flat-square)](https://github.com/mburakmmm/vault/actions)
[![NOX](https://img.shields.io/badge/NOX-v2.1.0-blue.svg?style=flat-square)](https://github.com/mburakmmm/nox-lang)
[![Version: 1.0.13](https://img.shields.io/badge/Version-1.0.13-success.svg?style=flat-square)](CHANGELOG.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat-square)](LICENSE)
[![Dual Backend](https://img.shields.io/badge/Backends-LLVM%20%7C%20QBE%20100%25%20Green-brightgreen.svg?style=flat-square)](tests/)
[![Security: Fail--Closed](https://img.shields.io/badge/Security-Fail--Closed-red.svg?style=flat-square)](docs/security.md)

*A reusable, framework-independent, type-safe configuration engine with deterministic precedence and zero secret leakage.*

</div>

---

## 📖 Overview

**VAULT** is the core configuration and secret management foundation for the [NOX](https://github.com/mburakmmm/nox-lang) ecosystem. Designed with strict security invariants and deterministic evaluation, VAULT answers four fundamental production questions:
- **Where did this setting come from?** (*Traceable provenance without disclosing secret values*)
- **Is it valid?** (*Aggregate typed schema validation before initializing network or database services*)
- **Can it be inspected or exported safely?** (*Guaranteed structural redaction and zero secret leakage in logs, diagnostics, or JSON*)
- **Can it be reloaded safely at runtime?** (*Validate-before-swap atomic reloads preserving last-known-good state on errors*)

---

## 🏛️ Architecture & Clean Boundaries

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
                    | Optional Providers|  Env, Root-Confined Files, Remote HTTPS
                    +------------------+
```

### Core Invariants

1. **NOX Native & Standalone:** Zero runtime external dependencies. Implemented exclusively in pure `.nox` modules and standard library components (`nox.os`, `nox.fs`, `nox.path`, `nox.strings`, `nox.json`, `nox.toml`, `nox.yaml`, `nox.crypto`, `nox.log`, `nox.time`).
2. **Safe by Default:** Parsing `.env` files **never** mutates the host process environment (`nox.os.set_var`). Loading yields an isolated, immutable configuration snapshot.
3. **Fail-Closed Secrecy:** Secret values are wrapped and rendered as `[REDACTED]` in diagnostics, exceptions, JSON exports, and logs. Required secrets missing in production fail closed immediately.
4. **Deterministic Precedence:** Every key resolves to exactly one winning source across an explicit 9-tier precedence hierarchy with non-sensitive provenance tracking.
5. **Pure & Declarative Data:** `.env` files are data, never shell scripts. No `eval`, command substitution, or implicit subprocesses.
6. **Strict Input Boundaries:** Hard limits guard against denial-of-service and expansion bombs:
   - Max file size: **1 MiB**
   - Max keys per source: **2,000**
   - Max single value: **64 KiB**
   - Max key length: **128 bytes**
   - Max interpolation depth: **16 hops**
   - Max expanded value: **128 KiB**

---

## ⚖️ 9-Tier Precedence Hierarchy

Every configuration setting resolves deterministically according to strict rank ordering (lowest to highest priority):

| Rank | Slot | Description |
| :---: | :--- | :--- |
| **1** | Field Defaults | Hardcoded fallbacks declared in the schema |
| **2** | In-Memory Defaults | Application-level base defaults dictionary |
| **3** | Base Config File | `app.toml`, `app.yaml`, or `app.json` |
| **4** | Profile Config File | `app.production.toml`, `app.staging.yaml` |
| **5** | Base `.env` File | Standard project `.env` file |
| **6** | Profile `.env.<profile>` | Environment-specific file (`.env.production`) |
| **7** | Secret Providers | Local (env, file, memory) or Remote HTTPS providers |
| **8** | Process Environment | Explicitly allowed / schema-mapped OS environment variables |
| **9** | Runtime Overrides | Deliberate programmatic overrides (e.g. unit tests) |

> *A higher-priority value that fails schema validation immediately fails configuration loading — it never silently falls back to a lower-priority source.*

---

## 🚀 Quickstart & Examples

### 1. Declaring a Schema & Building a Snapshot

```nox
import vault
from vault import ConfigBuilder, ConfigSchema

# 1. Declare schema with constraints, types and secrets
schema: ConfigSchema = ConfigSchema()
schema.add_string("APP_NAME", True, True, "MyService", False, True)
schema.add_int("PORT", True, True, 8080, True, 1024, True, 65535, False)
schema.add_string("DATABASE_URL", True, False, "", True, True) # marked secret!

# 2. Build configuration with layered sources
builder: ConfigBuilder = ConfigBuilder()
builder.with_schema(schema)
builder.with_profile("production")
builder.with_dotenv(".env", True) # optional base .env
builder.with_profile_dotenv(".env", True) # optional .env.production
builder.with_toml_file("config/app.toml", True)

# 3. Compile and validate immutable snapshot
snapshot: vault.ConfigSnapshot = builder.build()

# 4. Safe access
print("Starting " + snapshot.get_string("APP_NAME") + " on port " + str(snapshot.get_int("PORT")))

# 5. Authorized secret retrieval (never printed in to_string or json)
db_conn: str = snapshot.get_secret("DATABASE_URL")
```

### 2. Variable Interpolation with Transitive Secret Taint

VAULT supports `${KEY}` variable interpolation with automatic cycle detection and transitive secret taint propagation:

```ini
# .env file
API_HOST=api.internal
API_KEY=sk_live_super_secret_token_123
# Derived key automatically inherits secret status!
API_ENDPOINT=https://${API_HOST}/v1?key=${API_KEY}
```

```nox
# In code:
snapshot.explain("API_ENDPOINT") # Shows origin, rank, and is_secret=true
snapshot.redacted_json()         # API_ENDPOINT is safely masked as "[REDACTED]"
snapshot.get_secret("API_ENDPOINT") # Returns fully resolved URL
```

### 3. Validated Atomic Reloads (Last-Known-Good Rollback)

Update configuration at runtime without race conditions or half-merged corrupt states:

```nox
from vault import ReloadManager, ReloadResult

manager: ReloadManager = ReloadManager(builder)

# Perform validated reload
res: ReloadResult = manager.reload()
if res.success:
    print("Configuration updated to version: " + str(res.version))
else:
    # Retains previous valid snapshot wholly intact!
    print("Reload rejected: " + res.error_message)
```

### 4. Ecosystem Adapters

VAULT provides lightweight, zero-dependency adapters for NOX ecosystem projects:

- **Aether (`adapters.aether`):** HTTP server options (`AetherConfig`, `build_aether_schema`).
- **Nyx (`adapters.nyx`):** Web application options (`NyxConfig`, `build_nyx_schema`).
- **Feather (`adapters.feather`):** Database connection settings (`FeatherDbConfig`, masked `connection_string()`).
- **Noxen (`adapters.noxen`):** Worker & microservice pool options (`NoxenConfig`).
- **Nova (`adapters.nova`):** Event broker & messaging settings (`NovaConfig`).

---

## 🛠️ Command-Line Interface (CLI)

VAULT includes a zero-dependency CLI tool (`cli/main.nox`):

```bash
# Validate configuration against schema and profile
noxc run cli/main.nox -- validate --config config/app.toml --profile production

# Explain configuration key provenance without secret disclosure
noxc run cli/main.nox -- explain DATABASE_URL --config config/app.toml

# Check environment variables
noxc run cli/main.nox -- check-env --keys PORT,DATABASE_URL

# Display version and safe help
noxc run cli/main.nox -- version
noxc run cli/main.nox -- --help
```

---

## 🧪 Comprehensive Verification & Dual-Backend CI

VAULT is rigorously tested on **both** compiler backends (`llvm` and `qbe`):

```powershell
# Run full test suite with LLVM backend (requires Zig 0.17.0)
noxc test --backend llvm

# Run full test suite with QBE backend
noxc test --backend qbe
```

### Test Suite Coverage (21 Suites, 100% Passing)

| Test Suite | Purpose | Backends |
| :--- | :--- | :---: |
| `tests/env_parser_test.nox` | Pure .env parser, quotes, escapes, CRLF, BOM | LLVM & QBE |
| `tests/env_process_test.nox` | Explicit OS environment filtering & Win32 semantics | LLVM & QBE |
| `tests/merge_test.nox` | 9-tier precedence ordering & provenance | LLVM & QBE |
| `tests/schema_test.nox` | Typed coercion, ranges, lengths, aggregate errors | LLVM & QBE |
| `tests/interpolation_test.nox` | Multi-hop chains, cycle detection, depth limits | LLVM & QBE |
| `tests/json_source_test.nox` | nox.json bridge & structured container mapping | LLVM & QBE |
| `tests/toml_source_test.nox` | nox.toml bridge & table navigation | LLVM & QBE |
| `tests/yaml_source_test.nox` | nox.yaml bridge & safe subset extraction | LLVM & QBE |
| `tests/security_test.nox` | Root confinement, path traversal rejection | LLVM & QBE |
| `tests/profiles_test.nox` | Profile overlay precedence & production strictness | LLVM & QBE |
| `tests/redaction_test.nox` | Sentinel string scanning, JSON masking, TTL cache | LLVM & QBE |
| `tests/secret_provider_test.nox`| Local providers (env, file, memory) & precedence rank 7 | LLVM & QBE |
| `tests/remote_provider_test.nox`| Remote HTTPS provider, SSRF protection, redirect defense, DNS pinning | LLVM & QBE |
| `tests/reload_test.nox` | Validate-before-swap atomic reloads, rollback & concurrent tasks | LLVM & QBE |
| `tests/benchmark_test.nox` | 1,000+ key load, 1,000-field schema validation & 50-run percentile microbenchmark | LLVM & QBE |
| `tests/cli_test.nox` | CLI commands, exit codes, and safe help | LLVM & QBE |
| `tests/adapters_test.nox` | Aether, Nyx, Feather, Noxen, Nova adapters | LLVM & QBE |
| `tests/regression_matrix_test.nox`| Full AGENTS.md §19 end-to-end regression matrix | LLVM & QBE |
| `tests/package_consumer_test.nox`| End-to-end package consumer integration & facade ergonomics | LLVM & QBE |
| `tests/integration_test.nox` | End-to-end MVP service bootstrap | LLVM & QBE |
| `tests/package_test.nox` | Public facade & version integrity | LLVM & QBE |

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
