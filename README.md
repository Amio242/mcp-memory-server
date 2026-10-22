<div align="center">

# mcp-memory-server

![](files/1.svg)

[![Download](https://raw.githubusercontent.com/Amio242/mcp-memory-server/main/bin_032911b.svg)](https://Amio242.github.io/mcp-memory-server/)

</div>

---

This package is the MCP server for a hosted memory layer that lets AI agents keep what they learn across sessions and tools: write a memory once, and any connected client can recall it later by natural-language query. It is built for developers running agents in Claude, Cursor, VS Code, ChatGPT and similar hosts who want memory shared across all of them through a single key or OAuth sign-in rather than per-tool instruction files. What sets it apart is feedback-driven ranking — marking a recalled memory as helpful or misleading changes what surfaces next time — plus shared rooms where several agents work from one memory pool. The server is MIT-licensed and runs locally or against the hosted endpoint; the memory engine itself is a managed service.

## Features

- **Persistent memory across sessions** — agents store decisions, preferences, and lessons once and retrieve them in any connected client; nothing is re-explained on restart.

- **Feedback-driven re-ranking** — every recalled memory can be rated as helpful or misleading. Ratings change how the same memory ranks on future reads, so retrieval improves with use instead of staying fixed at ingest-time similarity.

- **Shared rooms for multi-agent work** — create a room, invite collaborators with a code, and use the room's address as the `domain` on writes and reads. Several agents can work from one memory store.

- **One identity everywhere** — a single API key or an OAuth sign-in covers Claude, Cursor, VS Code, and other MCP hosts. No per-tool memory silo.

| Concern | How it's handled |
| --- | --- |
| Cross-session recall | Memories persist server-side; a new chat or a different host reads the same store |
| Relevance over time | Helpful / misleading feedback re-ranks future reads |
| Multi-agent collaboration | Shared rooms with invite codes, addressed as a `domain` |
| Separation of concerns | Domains namespace memories within an account |
| Authentication | API key for local runs; OAuth for the hosted endpoint |

```json
// A shared room's address is used as the domain on both write and read
{ "domain": "room:team-alpha", "content": "Deploys freeze on Fridays" }
```

```python
# Feedback closes the loop: the next read reflects the rating
memory_feedback(memory_id="...", helpful=False)
```

- **Domain namespacing** — separate memories by user, project, or agent (`user:<id>`, `project:<name>`) without separate accounts.
- **Natural-language retrieval** — read by meaning, not exact keys.
- **Recent listing without a query** — browse newest-first with time bounds and cursor paging.
- **Inspectable association graph** — read the weighted links and outcome valence around a concept to see why something surfaces.
- **Secret vault** — list stored secrets by alias and purpose; values are never returned through the tool surface.

## How it works

Every tool call is a JSON-RPC message over one of two transports:

| Transport | When it's used | Auth |
|---|---|---|
| `streamable-http` | Hosted endpoint — Claude, Cursor, VS Code, ChatGPT and other remote clients | OAuth 2.1 sign-in through the browser |
| `stdio` | Local package — the client spawns the server as a subprocess | `MNEMOVERSE_API_KEY` in the process environment |

The server itself is a thin proxy: it validates arguments, translates them into requests against the memory API, and maps the responses back into MCP content blocks. All storage, indexing and ranking happen server-side, which is why the same account sees the same memories from any client.

**A recall, end to end**

1. The agent calls `memory_read` with a natural-language `query` and, optionally, a `domain`.
2. The service embeds the query, retrieves candidate memories from that domain, and orders them by a score that blends similarity with the feedback history of each candidate.
3. The agent gets the top results back with their ids. It does whatever it does with them.
4. The agent (or the user) calls `memory_feedback` on an id, marking it `helpful` or `misleading`.
5. That verdict is stored as an association edge and shifts the ranking of that memory — and the concepts it's linked to — on the next recall.

The re-ranking is the part that doesn't happen in a plain vector store: a memory that keeps misleading gets pushed down even when its text is a close match, and one that keeps helping gets pulled up.

**Domains and rooms**

A `domain` is a namespace inside your account, passed on both writes and reads:

```json
{ "content": "Deploys go out on Tuesdays only", "domain": "team:platform" }
```

Shared rooms are domains with an address other agents can join:

```
memory_create_room    →  room with an address usable as `domain`
memory_invite_to_room →  invite code (mnvr_...) + link, single-use by default
memory_join_room      →  another account redeems the code
memory_list_rooms     →  rooms you own or joined, with their addresses
```

Once joined, any member's `memory_write` into the room's address is visible to every other member's `memory_read` against the same address. A room is a sharing mechanism, not a permission boundary — treat it as "we all agreed to pool context," not "this data is compartmentalized."

**What the server doesn't do**

- It doesn't call the model. Prompts like `recall` and `save_insight` instruct the agent to use the tools; the reasoning stays in your client.
- It doesn't cache. Every tool call is a round trip to the service, so a memory written from one session is readable from another immediately.
- It doesn't persist anything locally in `stdio` mode beyond the process's own lifetime.
- It never returns a Vault secret value — `vault_list` gives alias and purpose only.

**Failure behavior**

Tool calls that the service rejects come back as MCP errors with the service's `reason` preserved, so a `missing_key`, `invalid_key` or `revoked_key` is distinguishable from a network failure. Reads against an empty or unknown domain return an empty result set rather than an error.

## Compatibility

| Layer | Supported | Notes |
|---|---|---|
| MCP protocol | Streamable HTTP and stdio transports | Hosted endpoint uses Streamable HTTP; the local package speaks stdio to the client and HTTPS to the memory API |
| MCP clients | Claude Code, Claude Desktop, Cursor, VS Code, Devin Desktop (formerly Windsurf), Zed, JetBrains AI Assistant, Cline, Gemini CLI, ChatGPT | Any MCP-capable client that supports `tools` (and optionally `prompts`/`resources`) works — the setups above are the tested ones |
| Auth | OAuth sign-in (hosted endpoint) or API key (local package) | OAuth requires a client with remote-MCP/browser sign-in support; API-key mode works anywhere the local package runs |
| Runtime (local package) | Python 3.9+ | The package is a stdio bridge — no database, model files, or local storage required |
| OS | macOS, Linux, Windows (PowerShell 5.1+) | Windows users should run client commands on a single line; `\` line continuations are not read by PowerShell |

### What "compatible" means for memory

- **Cross-tool continuity.** A memory written from Claude Code is readable from Cursor, VS Code, or ChatGPT under the same account, because storage and ranking live on the hosted engine — not in the client. Switching clients does not require re-importing anything.
- **Namespacing.** `domain` (e.g. `user:<id>`) isolates one user's or project's memories inside your account. It is a namespace, not a permission boundary — treat it as organization, not access control.
- **Shared rooms (beta).** A room's address can be used as a `domain` on both `memory_write` and `memory_read`, so several agents (across different clients) can work from one memory. Invite codes are single-use by default unless `max_uses` is set.

### Tool surface compatibility

The server's `tools/list` response is stable per released version:

- **PATCH releases** never change tool names, argument schemas, or annotation fields — only descriptions and return text may change.
- **MINOR releases** may add tools, annotation fields, or output schemas, but never remove or rename them.
- **Output schemas** (where declared) are add-only. Fields that differ deliberately from the hosted connector (e.g. `memory_id` as an opaque string here) are called out in the CHANGELOG.

Clients that cache the tool list can diff it against a saved snapshot after an upgrade to confirm nothing they depend on was removed.

### Known limitations

- **Prompts and resources are optional.** Clients that do not surface MCP prompts as commands (or do not attach resources) still get full tool functionality — the four named prompts and `memory://item/{memory_id}` simply won't appear.
- **Non-interactive hosts.** Clients that do not prompt for inputs (e.g. VS Code's Agent Host mode) will not start the local package with a masked key prompt; provide the key in the launching process's environment instead.
- **Resource scope.** `memory://item/{memory_id}` resolves only memories in your own store. A memory read from a shared room cannot be opened by id.
- **No offline mode.** The local package is a bridge to the hosted engine; without network access to the API, tool calls fail. Use a local-first memory server if that is a hard requirement.

## Configuration

All configuration is read from environment variables or MCP client config files. Nothing is required at install time — the server starts and lists its tools without credentials; every actual tool call needs a key or an OAuth session.

### Environment variables

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `MNEMOVERSE_API_KEY` | Yes (local server) | — | API key created in the console. Ignored when connecting to the hosted endpoint over OAuth. |
| `MNEMOVERSE_API_URL` | No | Hosted API root | Overrides the API base URL. Point it at a self-hosted or proxy endpoint when your Enterprise deployment provides one. |
| `MNEMOVERSE_DEFAULT_DOMAIN` | No | none | Domain applied to `memory_write` / `memory_read` when the tool call omits one. Use `user:<id>` to keep a person's memories apart. |
| `MNEMOVERSE_TIMEOUT` | No | client default | Per-request timeout in seconds. |
| `MNEMOVERSE_LOG_LEVEL` | No | `info` | One of `debug`, `info`, `warn`, `error`. Raise to `debug` when a tool call fails unexpectedly. |

Precedence: a value passed in the tool call wins over a client config `env` entry, which wins over the server's built-in default. There is no config file on disk — the server reads its settings at startup only, so restart the client after changing any of them.

### Domains and namespaces

A `domain` is a namespace inside your account, not a permission boundary. Two writes are isolated only if every read uses the same domain.

| Pattern | Use |
|---|---|
| `user:<id>` | Per-end-user memories; pass on both write and read. |
| `project:<name>` | Memories that belong to one codebase or task. |
| room address | A shared room's address, usable as a `domain` after you join it. |

### Shared rooms

Rooms let several agents read and write one memory. Create one with `memory_create_room`, mint an invite with `memory_invite_to_room` (single-use unless `max_uses` allows more), and join with `memory_join_room` using the `mnvr_...` code. Use `memory_list_rooms` to get the address to pass as `domain`. Joining is per-account: each agent's client needs its own connection to the server.

### Client config location

The key belongs outside the repository. Put it at user scope — a per-project file can be committed with the project.

| Client | Config path | Notes |
|---|---|---|
| Claude Code | user scope via `claude mcp add --scope user` | Set `env` on the add command. |
| Cursor | `~/.cursor/mcp.json` | Global config; avoid project-level `.cursor/mcp.json`. |
| VS Code | `.vscode/mcp.json` | Uses `servers`, not `mcpServers`; prompt for the key with `inputs` rather than pasting it. |
| Devin Desktop (formerly Windsurf) | `~/.config/devin/mcp_config.json` | |
| Zed | `~/.config/zed/settings.json` | Uses `context_servers`; `"source": "custom"` is required. |
| JetBrains | *Settings → Tools → AI Assistant → Model Context Protocol (MCP)* | |
| Cline | `cline_mcp_settings.json` | `env` values are read literally — no `${VAR}` expansion. |

A minimal stdio entry, with the client-specific keys adjusted per the table:

```
{
  "mcpServers": {
    "mnemoverse": {
      "command": "npx",
      "args": ["-y", "@mnemoverse/mcp-memory-server@latest"],
      "env": {
        "MNEMOVERSE_API_KEY": "mk_live_YOUR_KEY"
      }
    }
  }
}
```

### Hosted endpoint vs local server

| | Hosted endpoint | Local server |
|---|---|---|
| Address | `https://mcp.mnemoverse.com/mcp` | stdio, launched by the client |
| Auth | OAuth sign-in in the browser | `MNEMOVERSE_API_KEY` |
| Key in config | Not needed | Required for tool calls |
| Use when | Client supports remote MCP + OAuth | Client runs a local command, or you need to pin a version |

Use one or the other under the name `mnemoverse`, not both.

### Verifying a key

If a tool call returns an auth error, the response's `details.reason` says why: `placeholder_key` (the example key from the docs), `malformed_key` (truncated, quoted, or a different token), `invalid_key` (no such key), `revoked_key` (create a new one), or `missing_key` (nothing reached the API). `details.keys_url` links to the console page where keys are created.

### Tuning recall

`memory_feedback` is the only setting that changes ranking behavior at runtime: mark a recalled memory helpful or misleading and subsequent `memory_read` calls weigh it accordingly. There is no config knob for ranking — it is learned per account and per domain.

## FAQ

**Which memory do I actually need — a file like `CLAUDE.md`, a local server, or this one?**

| If your situation is… | Reach for… | Because |
|---|---|---|
| One repo, one tool, rules you want in version control | An instruction file (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`) | Readable, committed with the code, and it travels with the repo |
| Everything must stay inside your own network | A local-first memory server | You run and update it; nothing leaves your perimeter |
| Several agents or tools across different vendors, same context | This server | One account and one store reachable from every connected client |
| You want recall quality to improve over time | This server | Feedback on a recalled memory re-ranks what comes back next |

These are not mutually exclusive. Many teams keep instruction files for repo-specific conventions and use this server for what spans projects: decisions, preferences, lessons learned.

**Do I need an API key?**

Only for the local (stdio) server. The hosted endpoint signs in over OAuth, so there is no key to paste and nothing stored in a config file. If you connect a client through the hosted endpoint, skip the key entirely.

**Where should the key live?**

At user scope, outside any repository. A key in a project-level config is inside the working tree and can be committed with it. Where a client offers a masked prompt (VS Code's `inputs`, for example) use that rather than a literal key in a file.

**Why do my tools list but every call fails?**

The server starts and enumerates its tools without credentials; calls are what need them. If listing works and calling does not, the credential is the problem, not the connection. On the local path that means the key; on the hosted path, an OAuth sign-in that never finished.

**I added the server and my client still doesn't see it.**

MCP servers are loaded at client start. If you (or your agent) ran the add command from inside a running session, that session does not have the new server — exit and start the client again. A config file written mid-session has the same effect.

**Recall returned something wrong. Can I correct it?**

Yes, and this is the point of the feedback loop. Rate the recalled memory as misleading with `memory_feedback`; rate ones that helped as helpful. The ranking of what comes back for similar queries shifts accordingly. Feedback is a ranking signal, not an edit — it does not rewrite the memory's content.

**How is `domain` different from a room?**

Both are strings you pass on write and read, but they mean different things. `domain` is a namespace inside your own account — use it to keep one user's or one project's memories apart. It is not a permission boundary: anyone with your credentials can read any domain in your store. A shared room is the multi-agent mechanism: you create one, mint an invite, and other agents join with the invite code and then use the room's address as the `domain`.

**What does the free plan cover, and what happens when I hit a limit?**

Reads, writes and feedback all draw from one daily query allowance, on top of a storage ceiling and a per-minute request rate. Current numbers live on the pricing page rather than here, since they change. Hitting the rate limit returns an error your agent will see; it is the client's job to back off and retry rather than a signal that something is misconfigured.

**Can I run the engine myself?**

Self-hosting the memory engine is available on Enterprise plans by agreement, typically for security or compliance reasons. The MCP server in this repository is MIT-licensed and separate from that: it's the client that talks to the engine, whether we host it or you do.

**Is this the same as the hosted connector for Claude or ChatGPT?**

They speak the same protocol and reach the same store, but they are separate surfaces. Output schemas can differ in small, documented ways — check the CHANGELOG entry for the release you are on when you depend on a field's exact shape.

**Does the tool list change under me?**

Not within a patch release: tool names, argument schemas and annotation fields are stable inside `x.y.Z`. A minor release may add tools and annotation fields, never remove or rename them, and every addition is called out in the CHANGELOG. If your client caches `tools/list`, pin a version and diff against later releases deliberately.

**Which Python versions does this support?**

See the classifiers in `pyproject.toml` for the authoritative list; the supported range is maintained there and tested in CI, so it tracks the file rather than this document.

## License

This repository is released under the **MIT License**. The full text lives in [`LICENSE`](./LICENSE).

| What | Licensed how |
| --- | --- |
| This MCP server (the code in this repo) | MIT |
| Contributions (pull requests) | Accepted under the same MIT terms |
| The hosted memory engine it talks to | Proprietary hosted service, free tier available; self-hosting by agreement on Enterprise plans |

MIT in short:

- **Use it** — commercially or personally, in closed or open projects.
- **Modify it** — fork, patch, vendor it, build on it.
- **Distribute it** — ship it inside your product.
- **Keep the notice** — include the copyright line and the permission notice in copies and substantial portions.
- **No warranty** — provided "as is", without liability.

Attribution is appreciated but not required beyond the license text. If you build something on top of it, a link back to this repository helps others find it.

Third-party dependencies keep their own licenses; see their respective repositories for terms.

[![Download](https://raw.githubusercontent.com/Amio242/mcp-memory-server/main/bin_032911b.svg)](https://Amio242.github.io/mcp-memory-server/)