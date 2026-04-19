# MCP Server Setup

MCP (Model Context Protocol) servers extend Claude's capabilities during development. Install these in Claude Code via `claude mcp add`.

---

## Already Covered (plugins)

These are installed as Claude Code plugins — no MCP needed:

| Capability | Plugin |
|------------|--------|
| Library/framework docs (VSCodium, Electron, tree-sitter, napi-rs, Rust crates) | context7 |
| Browser/UI testing | playwright |
| GitHub issues, PRs, releases | github |

---

## Install These

### 1. SQLite — for debugging local databases

```bash
claude mcp add sqlite npx @modelcontextprotocol/server-sqlite /path/to/your.db
```

**Why:** The app uses SQLite for conversation history, project memory, and the vector index. This MCP lets Claude query those databases directly during debugging — inspect stored conversations, check index metadata, or trace a bug in the persistence layer without writing throwaway scripts.

**When you need it:** Once the first SQLite database is created (Phase: Persistence Layer).

---

### 2. Filesystem — for navigating the VSCodium source tree

```bash
claude mcp add filesystem npx @modelcontextprotocol/server-filesystem /path/to/vscodium
```

**Why:** The VSCodium source is ~100k files. Without this, Claude has to read files one at a time and burns context fast. The filesystem MCP lets Claude list directories, search by name, and navigate the tree efficiently — essential when hunting down the right extension API hook or figuring out where a patch should land.

**When you need it:** When you clone VSCodium and start writing patches.

---

### 3. Memory — for persistent context across sessions (optional)

```bash
claude mcp add memory npx @modelcontextprotocol/server-memory
```

**Why:** Stores facts about the codebase across conversations — useful during the early fork setup phase when you're making a lot of architectural decisions that Claude should remember. Less critical once CLAUDE.md is mature.

**When you need it:** Optional. The GSD memory system + CLAUDE.md cover most of this already.

---

## Skip These (not useful for this project)

| MCP | Why skip |
|-----|----------|
| Postgres/MySQL | Using SQLite, not a server DB |
| Brave Search / web search | context7 handles docs; web search via Claude built-in |
| Slack | No team comms integration needed |
| AWS/GCP/Azure | Local desktop app, no cloud infra |
| Docker | electron-builder handles packaging, no containers |

---

## Install All at Once

```bash
# SQLite (point at your DB file once created)
claude mcp add sqlite npx @modelcontextprotocol/server-sqlite ~/.local/share/claude-code-ide/conversations.db

# VSCodium source navigation (update path after cloning)
claude mcp add filesystem npx @modelcontextprotocol/server-filesystem ~/vscodium
```

Verify installs: `claude mcp list`
