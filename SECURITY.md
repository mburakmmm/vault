# Security Policy

## Reporting Security Issues

VAULT takes security and secret isolation with utmost priority. If you discover a vulnerability or potential secret disclosure issue:

1. **Do not open a public issue.**
2. Send a detailed report to the repository owner or security contact.
3. Include reproduction steps without transmitting real secrets (use obvious dummy placeholders like `VAULT_TEST_SECRET_DO_NOT_PRINT`).

## Core Security Invariants

- **Zero Secret Leakage:** Values marked with `secret=true` are strictly redacted as `[REDACTED]` from string conversions, error messages, debug logs, `to_json` exports, and exception traces.
- **Fail-Closed by Default:** In strict and production modes, missing required secrets cause an immediate configuration halt; they never silently fall back to weaker sources or default values.
- **No Unsafe Execution:** `.env` files are pure declarative data. No shell execution, command substitution (`$(...)`), backticks, or subprocess evaluation is ever executed.
- **Confined File Loading:** Root-restricted file lookup verifies canonical paths using path-segment awareness, explicitly rejecting directory traversal (`../`) and symlink escapes.
- **No Automatic Network Access:** Remote secret providers are opt-in, explicitly configured, and HTTPS-only. Startup never queries local network, instance metadata, or cloud credential endpoints.
- **Bounded Resource Consumption:** Hard ceilings on configuration size (1 MiB), key count (2,000), single value size (64 KiB), key length (128 bytes), and interpolation depth (16) protect against denial of service and recursion bombs.
