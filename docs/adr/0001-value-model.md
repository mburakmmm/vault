# ADR 0001: Value Model, Representation & Presence Semantics

## Status
Accepted

## Context
VAULT must represent configuration loaded from diverse sources: `.env` files (pure key-value strings), process environment variables, structured documents (JSON, TOML, YAML), in-memory dictionaries, and secret providers.

We need a clear architectural decision on:
1. Internal key representation (flattened dotted path vs nested tree).
2. Typed representation of values.
3. Semantics of missing, null, and empty string states.

## Decision

### 1. Key Canonicalization
- Internal canonical keys will use canonical **dotted paths** (e.g. `database.host`, `server.port`).
- For flat sources like `.env` and environment variables, mapping rules are defined:
  - Explicit mapping declarations (`DATABASE_HOST` ↔ `database.host`).
  - Standard normalized mapping (e.g. uppercase to lowercase with double-underscore for nesting `APP_DATABASE__HOST` -> `database.host`).
- Dotted keys in literal source keys are rejected unless explicitly allowed by source configuration to prevent injection attacks.

### 2. Value Representation (`ConfigValue`)
- Instead of normalizing all values into plain strings, VAULT preserves source types:
  - `StringValue(s: str)`
  - `IntegerValue(i: int)`
  - `FloatValue(f: float)`
  - `BooleanValue(b: bool)`
  - `ArrayValue(items: list[ConfigValue])`
  - `ObjectValue(entries: dict[str, ConfigValue])`
  - `NullValue()`
- Secret classification is a **metadata attribute** attached to `ConfigEntry`, not a distinct primitive scalar that discards the underlying type.

### 3. Presence Semantics
Four distinct presence states are maintained:
- **`MISSING`**: No source provided any value for this key. Triggers default value if available, or validation failure if required.
- **`NULL`**: An explicit null value provided by a structured source (JSON `null`, YAML `null`/`~`). Distinct from missing; does not trigger default value unless explicitly configured.
- **`EMPTY_STRING`**: `""` explicitly provided. Valid as a string value unless schema constraint `non_blank=true` is set.
- **`SUPPLIED`**: Valid typed value supplied by a source.

### 4. Immutability
`ConfigSnapshot` is an immutable, frozen data structure once published. Builders construct candidate snapshots in isolated buffers; snapshots exposed to readers cannot be mutated in place.

## Consequences
- Preserves native types from TOML, JSON, and YAML without premature string serialization.
- Aggregated validation can strictly differentiate between an omitted setting and an explicitly nullified setting.
- Redaction and secret classification propagate across clones, merges, and variable interpolation without losing underlying type metadata.
