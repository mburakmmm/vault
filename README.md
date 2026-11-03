# VAULT

> **VAULT** is a configuration, environment, and secret management library for the [NOX](https://github.com/mburakmmm/nox-lang) programming language.

## Design Principles

- **Safe by Default:** Reading `.env` files never mutates process environment variables (`nox.os.set_var`).
- **Fail-Closed:** In strict/production mode, missing required secrets fail immediately instead of silently falling back to insecure defaults.
- **Zero Secret Leakage:** Values classified as secrets are automatically redacted (`[REDACTED]`) from logs, string representations, error reports, and inspection output.
- **Deterministic Precedence:** Multi-source configuration (defaults, files, dotenv, environment, overrides) resolves in a strictly ordered, traceable, non-secret provenance tree.
- **Bounded Inputs:** Hard limits are enforced on file size, key count, value length, and interpolation depth to protect against denial-of-service.

## Compiler Compatibility

Pinned compiler: **NOX v2.0.123** (Hosted profile).  
Detailed stdlib signatures and verification results are documented in [docs/compatibility.md](docs/compatibility.md).

## Project Roadmap

See [todo.md](todo.md) for full phase-by-phase implementation progress and acceptance criteria.
