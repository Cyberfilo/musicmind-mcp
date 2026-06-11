# Security Policy

## Reporting a vulnerability

Please report security issues privately by emailing **filippo@menghi.dev** (the maintainer) with the subject line `musicmind-mcp security report`. Please **do not** open a public GitHub issue for a suspected vulnerability — open a [GitHub Security Advisory](https://github.com/Cyberfilo/musicmind-mcp/security/advisories/new) instead.

We aim to acknowledge a report within 7 days and disclose a fix within 30 days for confirmed issues.

## Threat model

`musicmind-mcp` sits between an LLM client (Claude Desktop, Claude Code, Cursor, etc.) and the Apple Music Web API. It caches the user's library and listening history in a local SQLite database to support taste profiling and recommendations.

### In-scope (what we defend against)

- **Untrusted MCP client input.** All tool inputs are validated against the registered JSON schemas via `pydantic`. The taste-profile and recommendation tools never invoke `eval`, `exec`, `pickle`, or `subprocess` on caller-supplied strings.
- **Apple Music token misuse.** The developer and user tokens are read from environment variables only (`APPLE_MUSIC_DEVELOPER_TOKEN`, `APPLE_MUSIC_USER_TOKEN`); they are never logged, never echoed to MCP responses, and never written to the local cache. The server refuses to start if either token is unset.
- **Local cache poisoning.** The SQLite cache is keyed by Apple-Music-issued `catalogId` strings, which are URL-safe and validated before use. Cache writes use parameterized SQL only.
- **Prompt injection via track / album / playlist text.** Apple Music metadata (titles, artist names, editorial blurbs) flows back through the LLM. Each metadata field is bounded to a maximum length and trimmed of control characters before being returned in MCP tool responses, limiting the surface for indirect prompt injection from compromised editorial content.
- **Cross-user data leakage.** A single server process is scoped to a single Apple Music user token. The cache database file is created with `0600` permissions and lives under `$HOME/.musicmind/cache.db` by default. There is no multi-tenant code path.

### Out-of-scope

- **Compromise of the host machine, the Claude client, or the Apple Music account itself.** If the host is compromised, the SQLite cache and the environment-variable tokens are reachable by the attacker; this is the same trust model as any local MCP server.
- **Apple's MusicKit API correctness or availability.** We forward through to Apple Music; we do not re-implement its authorization checks.
- **Network adversaries between the host and `api.music.apple.com`.** TLS is enforced by the `httpx` client; if your CA store is compromised, you have larger problems than this server.
- **DoS by malicious LLM-generated tool calls.** A misbehaving LLM can issue a flood of valid tool calls. We do not rate-limit beyond Apple's own per-token quotas.

### Tool annotations

Per the MCP spec, every tool registers `readOnlyHint`, `destructiveHint`, and `idempotentHint`. Tools that mutate user state (e.g. `create_playlist`, `add_to_library`) carry `destructiveHint: true` so spec-compliant clients can prompt for confirmation. Read-only tools (`get_track`, `taste_profile`, `recommendations`) carry `readOnlyHint: true`.

### Token rotation guidance

The Apple Music user token is short-lived (six months by default). Rotate it by re-running the MusicKit login flow on your client device and updating the `APPLE_MUSIC_USER_TOKEN` environment variable. The server detects 401/403 responses and exits with a clear message so the orchestrator can restart with fresh credentials.
