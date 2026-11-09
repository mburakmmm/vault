# VAULT API Reference Manual (v1.0.0)

`vault` is a reusable, framework-independent NOX 2.x library for type-safe configuration, environment management, and secret isolation.

---

## Table of Contents

- [Package Entry Point](#package-entry-point)
- [Core Builders & Snapshots](#core-builders--snapshots)
  - [`ConfigBuilder`](#configbuilder)
  - [`ConfigSnapshot`](#configsnapshot)
  - [`ConfigEntry` & `ConfigValue`](#configentry--configvalue)
- [Schema Definition & Validation](#schema-definition--validation)
  - [`ConfigSchema`](#configschema)
  - [`Field`](#field)
  - [`ValidationReport`](#validationreport)
- [Configuration Sources](#configuration-sources)
  - [`DotenvSource`](#dotenvsource)
  - [`ProcessEnvSource`](#processenvsource)
  - [`JsonSource`, `TomlSource`, `YamlSource`](#jsonsource-tomlsource-yamlsource)
  - [`MemorySource`](#memorysource)
- [Secret Management & Redaction](#secret-management--redaction)
  - [`SecretValue`](#secretvalue)
  - [`redact_string`](#redact_string)
  - [`SecretCache`](#secretcache)
- [Secret Providers](#secret-providers)
  - [`SecretProvider`](#secretprovider)
  - [`MemorySecretProvider`](#memorysecretprovider)
  - [`EnvSecretProvider`](#envsecretprovider)
  - [`FileSecretProvider`](#filesecretprovider)
  - [`RemoteHttpSecretProvider`](#remotehttpsecretprovider)
- [Atomic Reload](#atomic-reload)
  - [`ReloadManager`](#reloadmanager)
  - [`ReloadResult`](#reloadresult)
- [Command-Line Interface](#command-line-interface)
  - [`run_cli`](#run_cli)
- [Ecosystem Adapters](#ecosystem-adapters)
  - [Aether (`adapters.aether`)](#aether-adaptersaether)
  - [Nyx (`adapters.nyx`)](#nyx-adaptersnyx)
  - [Feather (`adapters.feather`)](#feather-adaptersfeather)
  - [Noxen (`adapters.noxen`)](#noxen-adaptersnoxen)
  - [Nova (`adapters.nova`)](#nova-adaptersnova)

---

## Package Entry Point

```nox
import vault
```

### `version() -> str`
Returns the current stable semantic version of the VAULT library (`"1.0.0"`).

---

## Core Builders & Snapshots

### `ConfigBuilder`
Fluent builder class used to construct an immutable, validated `ConfigSnapshot`.

```nox
from vault import ConfigBuilder
builder: ConfigBuilder = ConfigBuilder()
```

#### Methods

- **`with_schema(schema: ConfigSchema) -> ConfigBuilder`**  
  Attaches a schema definition against which all collected configuration entries will be validated and typed.
- **`with_profile(name: str) -> ConfigBuilder`**  
  Sets the active execution profile (e.g., `"development"`, `"test"`, `"staging"`, `"production"`).
- **`with_allowed_root(root: str) -> ConfigBuilder`**  
  Restricts all file system reads (JSON, TOML, YAML, secret files) to within the specified canonical root directory.
- **`with_source(source: ConfigSource) -> ConfigBuilder`**  
  Registers a generic configuration source into the merge queue.
- **`with_defaults(defaults: dict[str, str]) -> ConfigBuilder`**  
  Registers in-memory base defaults at precedence rank 2.
- **`with_json_file(path: str, optional: bool) -> ConfigBuilder`**  
  Loads a JSON configuration file at rank 3 (`base`) or rank 4 (`profile`).
- **`with_toml_file(path: str, optional: bool) -> ConfigBuilder`**  
  Loads a TOML configuration file at rank 3 (`base`) or rank 4 (`profile`).
- **`with_yaml_file(path: str, optional: bool) -> ConfigBuilder`**  
  Loads a YAML configuration file at rank 3 (`base`) or rank 4 (`profile`).
- **`with_dotenv(path: str, optional: bool) -> ConfigBuilder`**  
  Loads a `.env` file at rank 5.
- **`with_profile_dotenv(base_path: str, optional: bool) -> ConfigBuilder`**  
  Loads a profile-specific `.env.<profile>` file at rank 6.
- **`with_secret_provider(provider: SecretProvider, keys: list[str], fail_closed: bool) -> ConfigBuilder`**  
  Binds a secret provider for the requested keys at precedence rank 7.
- **`with_process_env(allowed_keys: list[str], mapping: dict[str, str]) -> ConfigBuilder`**  
  Binds explicitly allowed OS environment variables at precedence rank 8.
- **`with_override(key: str, val: str) -> ConfigBuilder`**  
  Sets an explicit runtime/test override at precedence rank 9.
- **`set_production(is_prod: bool) -> ConfigBuilder`**  
  Enforces strict production-mode checks (no default passwords, fail-closed required secrets).
- **`build() -> ConfigSnapshot`**  
  Merges all sources in strict rank order, resolves `${KEY}` interpolations with cycle detection, validates against the schema, and returns a frozen, immutable `ConfigSnapshot`. Raises `ConfigError` on any validation, format, cycle, or constraint failure.

---

### `ConfigSnapshot`
An immutable, frozen view of resolved configuration settings.

#### Methods

- **`has(key: str) -> bool`**: Returns `true` if the key exists and has a non-missing value.
- **`is_null(key: str) -> bool`**: Returns `true` if the key was explicitly set to null.
- **`get_string(key: str) -> str`**: Returns the key's value as a string.
- **`get_int(key: str) -> int`**: Returns the key's value coerced to an integer.
- **`get_float(key: str) -> float`**: Returns the key's value coerced to a float.
- **`get_bool(key: str) -> bool`**: Returns the key's value coerced to a boolean (`true`/`false`, `1`/`0`, `"yes"`/`"no"`).
- **`get_list(key: str) -> list[str]`**: Returns an array of strings.
- **`get_secret(key: str) -> str`**: Explicit, authorized retrieval of secret values.
- **`get_secret_value(key: str) -> SecretValue`**: Retrieves a `SecretValue` wrapper whose string representation is masked.
- **`explain(key: str) -> str`**: Returns provenance metadata (winning source, rank, profile, is_secret) **without** disclosing secret values.
- **`redacted_json() -> str`**: Returns a deterministic JSON string with all secrets masked as `"[REDACTED]"`.
- **`version() -> str`**: Returns the snapshot version identifier (e.g. `"v1"`, `"v2"`).

---

## Schema Definition & Validation

### `ConfigSchema`

```nox
from vault import ConfigSchema
schema: ConfigSchema = ConfigSchema()
```

- **`add_string(key, required, has_default, default_val, secret, non_blank)`**
- **`add_int(key, required, has_default, default_val, has_min, min_val, has_max, max_val, secret)`**
- **`add_float(key, required, has_default, default_val, has_min, min_val, has_max, max_val, secret)`**
- **`add_bool(key, required, has_default, default_val, secret)`**
- **`add_string_list(key, required, secret)`**

---

## Ecosystem Adapters

### Aether (`adapters.aether`)
```nox
from adapters.aether import AetherConfig, build_aether_schema, extract_aether_config

schema: ConfigSchema = build_aether_schema("http")
# After building snapshot:
cfg: AetherConfig = extract_aether_config(snapshot, "http")
```

### Feather (`adapters.feather`)
```nox
from adapters.feather import FeatherDbConfig, build_feather_schema, extract_feather_config

schema: ConfigSchema = build_feather_schema("db")
cfg: FeatherDbConfig = extract_feather_config(snapshot, "db")

# Masked connection string for logging
print("Connecting: " + cfg.to_string()) # postgresql://user:[REDACTED]@host:5432/db

# Unmasked for driver connection
driver.connect(cfg.connection_string(False))
```
