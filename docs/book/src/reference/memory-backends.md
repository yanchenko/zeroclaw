# Memory backends

ZeroClaw selects an install-wide storage instance with `memory.backend` and
binds each agent to a backend kind in `[agents.<alias>.memory]`. Shared stores
carry a stable agent UUID on every row; Markdown stores use per-agent files.

| Backend | Canonical storage | External service | Notes |
|---|---|---|---|
| `sqlite` | SQLite | no | Default; hybrid keyword and vector recall. |
| `markdown` | Markdown files | no | Human-readable, per-agent files. |
| `postgres` | PostgreSQL | yes | Optional pgvector support. |
| `qdrant` | Qdrant | yes | Vector-store payloads carry agent attribution. |
| `lucid` | SQLite | Lucid CLI | Best-effort derived-context enrichment. |
| `shodh` | SQLite | Shodh Memory HTTP API | Best-effort semantic enrichment with authenticated, agent-isolated remote users. |
| `none` | none | no | Disables persistent memory. |

## Shodh Memory

The Shodh backend is local-first. A successful write is committed to SQLite
before ZeroClaw attempts the HTTP mirror. Recall searches SQLite first and asks
Shodh only when the local result count is below the configured threshold. A
Shodh timeout or error therefore reduces semantic enrichment but does not make
local memory unavailable.

Configure a typed storage alias and point `memory.backend` at it:

```toml
[memory]
backend = "shodh.semantic"

[storage.shodh.semantic]
base_url = "http://127.0.0.1:3030"
api_key = "replace-with-the-shodh-api-key"
recall_timeout_ms = 2000
store_timeout_ms = 5000
local_hit_threshold = 3
failure_cooldown_ms = 15000

[agents.default.memory]
backend = "shodh"
```

`api_key` is a classified encrypted-secret field. ZeroClaw sends it only in
the `X-API-Key` header and rejects URLs containing embedded credentials. Do not
put the key in the URL.

Each stable ZeroClaw agent UUID becomes a separate Shodh `user_id`. Cross-agent
recall makes one bounded request for each UUID already allowed by
`read_memory_from`; it cannot widen that allowlist. Session, namespace,
category, and ZeroClaw key metadata are mirrored as tags.

Remote recall never becomes authoritative. Every Shodh result is mapped back
to its SQLite key and agent UUID, and ZeroClaw returns the canonical local row.
If the local row was deleted, a stale remote result is discarded. Scoped
forget, session purge, and agent purge are also propagated to Shodh on a
best-effort basis.

Recent-only recall (an empty query or `*`) remains local because Shodh's useful
path is semantic query recall. Existing SQLite rows are not bulk-uploaded when
the backend is first enabled; subsequent writes and updates create the mirror.

Back up `data/memory/` as usual. That contains the authoritative ZeroClaw
memory. Back up the Shodh service separately only if preserving its derived
semantic index is operationally important.
