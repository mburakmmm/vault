# VAULT Production Recipes & Architectural Patterns

This guide provides tested architectural patterns for common configuration challenges in the NOX ecosystem.

---

## Recipe 1: 12-Factor Microservice Bootstrap

```nox
from vault import ConfigBuilder, ConfigSchema, ConfigSnapshot
import nox.os

def load_app_config() -> ConfigSnapshot:
    # 1. Declare schema
    schema: ConfigSchema = ConfigSchema()
    schema.add_string("APP_NAME", True, True, "PaymentService", False, True)
    schema.add_int("PORT", True, True, 8080, True, 1024, True, 65535, False)
    schema.add_string("DATABASE_URL", True, False, "", True, True)
    schema.add_string("REDIS_URL", True, True, "redis://127.0.0.1:6379", False, True)

    # 2. Determine profile from environment or default to development
    active_profile: str = "development"
    try:
        active_profile = nox.os.getenv("APP_ENV")
    except nox.os.OsError:
        pass

    # 3. Build layered configuration
    builder: ConfigBuilder = ConfigBuilder()
    builder.with_schema(schema)
    builder.with_profile(active_profile)
    builder.with_dotenv(".env", True)
    builder.with_profile_dotenv(".env", True)
    builder.with_json_file("config/app.json", True)

    # 4. Map authorized OS environment variables
    allowed_os_vars: list[str] = ["PORT", "DATABASE_URL"]
    os_mapping: dict[str, str] = {}
    builder.with_process_env(allowed_os_vars, os_mapping)

    return builder.build()
```

---

## Recipe 2: Secure Database Connection with Feather

Avoid exposing database passwords in application logs and tracebacks:

```nox
from vault import ConfigBuilder, ConfigSnapshot
from adapters.feather import FeatherDbConfig, build_feather_schema, extract_feather_config

def init_database() -> None:
    schema = build_feather_schema("db")

    builder = ConfigBuilder()
    builder.with_schema(schema)
    builder.with_dotenv(".env", False) # requires database settings

    snapshot: ConfigSnapshot = builder.build()
    db_cfg: FeatherDbConfig = extract_feather_config(snapshot, "db")

    # Safe logging: password is automatically masked as [REDACTED]
    print("Connecting to: " + db_cfg.to_string())

    # Raw connection: authorized access
    # feather_pool.connect(db_cfg.connection_string(False))
```

---

## Recipe 3: Zero-Downtime Hot Reloads with `ReloadManager`

Dynamically reload settings in long-running services without race conditions:

```nox
from vault import ConfigBuilder, ConfigSnapshot
from reload.manager import ReloadManager, ReloadResult

builder = ConfigBuilder()
# ... configure builder with sources ...

manager: ReloadManager = ReloadManager(builder)

def on_config_change_event() -> None:
    result: ReloadResult = manager.reload()
    if result.success:
        print("Configuration successfully swapped to version: " + str(result.version))
        new_snapshot: ConfigSnapshot = manager.snapshot()
        # apply new non-breaking settings
    else:
        # Invalid candidate was safely rolled back!
        print("Reload rejected: " + result.error_message)
```
