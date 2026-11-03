# ADR 0002: Source Precedence, Merging Semantics & Provenance Tracking

## Status
Accepted

## Context
VAULT loads configuration from multiple layered sources: schema defaults, in-memory definitions, base and profile files, `.env` files, environment variables, secret providers, and runtime overrides. A deterministic merge hierarchy must guarantee that:
1. Every key resolves to exactly one winning source.
2. Safe provenance is recorded without disclosing secret values.
3. Merging composite structures (objects, arrays) follows consistent rules without corrupting types.

## Decision

### 1. Deterministic Precedence Hierarchy
Precedence ranks are strictly ordered from lowest (1) to highest (9):
1. **Schema Defaults (Rank 1):** Static fallbacks defined in schema fields.
2. **Memory Defaults (Rank 2):** Application-level base configurations.
3. **Base Config File (Rank 3):** `vault.toml`, `vault.yaml`, `vault.json`.
4. **Profile Config File (Rank 4):** `vault.<profile>.toml`, `vault.<profile>.yaml`.
5. **Base `.env` File (Rank 5):** Standard project `.env`.
6. **Profile `.env.<profile>` (Rank 6):** Environment overlay (e.g. `.env.production`).
7. **Secret Providers (Rank 7):** Configured secret provider adapters for declared keys.
8. **Process Environment (Rank 8):** Allowed/mapped operating system environment variables.
9. **Runtime Overrides (Rank 9):** Explicit test/runtime builder overrides.

### 2. Container and Scalar Merging Rules
- **Objects (`VAL_OBJECT`):** Recursive deep merge. Keys in higher-priority objects override keys in lower-priority objects; non-overlapping keys are retained.
- **Arrays (`VAL_ARRAY`):** Replace semantics by default. Higher-priority array completely replaces lower-priority array; arrays are not implicitly concatenated.
- **Container vs Scalar Conflict:** Attempting to merge a container (object/array) over a scalar or vice-versa immediately raises a `CONFIG_TYPE_MISMATCH` configuration error.
- **Empty Strings:** An explicit empty string (`""`, `PRESENCE_EMPTY`) is a valid override value and does not revert to lower-priority defaults.

### 3. Safe Provenance (`explain`)
- `explain(key)` outputs: `key`, `presence`, `source_kind`, `source_id`, `rank`, `is_secret`.
- **Invariance:** Secret values and raw configuration payloads are NEVER rendered in provenance descriptions.

## Consequences
- Guarantees 100% deterministic winning values regardless of filesystem read order or execution environment.
- Fails closed on structural type conflicts between layers.
- Diagnostic tools can inspect where settings originate without leaking credentials in logs or traces.
