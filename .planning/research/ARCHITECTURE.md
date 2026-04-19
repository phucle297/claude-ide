# Architecture Research — Claude Code IDE

**Researched:** 2026-04-19
**Confidence:** HIGH (Cursor internals verified via reverse engineering, VS Code architecture via official docs, Anthropic auth via official Claude Code docs)

---

## Component Overview

The system is composed of six major components with clear ownership boundaries:

| Component | Runtime | Responsibility |
|-----------|---------|---------------|
| **VS Code Fork Shell** | Electron (Node.js + Chromium) | Editor UI, command palette, layout, tabs, settings |
| **AI Panel Extension** | Extension Host (Node.js) | Chat UI, context picker, diff viewer, accept/reject |
| **Rust Indexer** | Native process via napi-rs | File watching, tree-sitter parsing, embedding generation, vector search |
| **Auth Manager** | Extension Host (Node.js) | OAuth PKCE flow, API key storage, token refresh |
| **Persistence Layer** | SQLite (via better-sqlite3 or rusqlite) | Conversation history, project memory, index metadata |
| **Claude API Client** | Extension Host (Node.js) | Assembles context, calls Anthropic API, streams responses |

**What talks to what:**

```
Chromium Renderer (UI)
    ↕ Electron IPC (ipcMain / ipcRenderer)
Extension Host (Node.js)
    ↕ napi-rs synchronous or async calls
Rust Indexer (native .node addon)
    ↕ reads/writes
SQLite databases (local disk)

Extension Host → HTTPS → Anthropic API
Extension Host → macOS Keychain / credentials.json → stored tokens
```

The Rust indexer does NOT run as a separate OS process — it runs as a native Node.js addon (napi-rs compiled `.node` file) inside the Extension Host. This avoids the 2-4ms IPC serialization overhead per call and eliminates sidecar lifecycle complexity, while still giving Rust full access to CPU and threading.

---

## Data Flow

### End-to-end: User asks a question → Claude responds with context

```
1. User types query in AI Panel chat input (Chromium renderer)
2. Renderer → ipcRenderer.invoke("ai:query", {text, @mentions}) → Extension Host
3. Extension Host: Context Manager assembles context budget
   a. Pull conversation history from SQLite (last N turns)
   b. Call Rust Indexer: semantic_search(query, top_k=20) → ranked chunk list
   c. Apply hierarchical budget: current file (2K) → imported files (3K)
      → semantically relevant (5K) → project structure (1K) → history (4K)
   d. Serialize to messages array for Anthropic API
4. Auth Manager provides bearer token (refresh if expired)
5. Claude API Client: POST /v1/messages with stream=true
6. Streaming tokens forwarded back to AI Panel renderer
7. On stream complete: persist full exchange to SQLite conversation DB
8. UI re-renders with final response + diff/edit suggestions
```

### Codebase indexing on project open

```
1. Extension Host detects workspace open event
2. Dispatch: start_indexing(workspacePath) → Rust Indexer
3. Rust Indexer:
   a. Walk directory tree (ignore .gitignore patterns)
   b. Hash each file (SHA-256); compare against stored hashes in SQLite index DB
   c. For new/changed files: parse with tree-sitter → extract functions, classes, modules
   d. Chunk into 100-500 token semantic units with 50-token overlap
   e. Batch-embed chunks (local model via fastembed-rs, or Anthropic embeddings API)
   f. Upsert vectors into local vector store (LanceDB or hnswlib via Rust)
   g. Persist chunk metadata (file path, line range, hash) to SQLite
4. File watcher (notify crate) monitors changes; triggers incremental re-index
5. Extension Host receives index_ready event; Context Manager is now query-capable
```

---

## Process Architecture

VS Code's native multi-process model is inherited:

```
┌─────────────────────────────────────────────────────────┐
│  Main Process (Electron main)                           │
│  - Window lifecycle, app menu, OS integration           │
│  - Spawns all child processes                           │
└───────────┬──────────────────────┬──────────────────────┘
            │ Electron IPC         │ Electron IPC
┌───────────▼──────────┐  ┌───────▼──────────────────────┐
│  Renderer Process    │  │  Extension Host Process      │
│  (Chromium)          │  │  (Node.js, sandboxed)        │
│  - Editor UI         │  │  - VS Code API surface       │
│  - AI Panel UI       │  │  - AI Panel Extension        │
│  - Diff viewer       │  │  - Auth Manager              │
│  - Context picker    │  │  - Claude API Client         │
│                      │  │  - Rust Indexer (.node)      │
└──────────────────────┘  └──────────────┬───────────────┘
                                         │ SQLite (file I/O)
                          ┌──────────────▼───────────────┐
                          │  Local Disk Storage          │
                          │  - conversation.db           │
                          │  - index.db + vector store   │
                          │  - credentials.json          │
                          └──────────────────────────────┘
```

**Why not a separate Rust process?** A sidecar OS process adds 2-4ms serialization overhead per call and requires process lifecycle management (spawn, health-check, restart). napi-rs native addons eliminate this: the `.node` file loads into the Extension Host's Node.js runtime, Rust code runs on native threads, and JS/Rust calls cross the FFI boundary in microseconds. This is the same pattern used by tools like SWC, Biome, and Parcel.

**Why Extension Host, not Main Process?** VS Code's architecture strongly discourages long-running work in the Main Process (it blocks the window). Extension Host is the correct home for all custom logic.

---

## Key Subsystems

### Codebase Indexer

**Technology:** Rust, compiled as napi-rs native addon (`.node` file)

**Pipeline:**

```
File Walker (ignore crate)
    → Hash comparison (incremental — skip unchanged)
    → tree-sitter parser (Rust bindings: tree-sitter-rust, tree-sitter-typescript, etc.)
    → AST chunker (functions, classes, top-level blocks; 100-500 token target)
    → Embedder (fastembed-rs for local; or HTTP call to Anthropic embeddings API)
    → Vector store upsert (LanceDB embedded — file-based, zero config)
    → SQLite metadata write (chunk → file path, line range, content hash)
```

**File watching:** `notify` Rust crate (cross-platform: FSEvents on macOS, inotify on Linux, ReadDirectoryChangesW on Windows). On change event: re-hash → diff → partial re-index.

**Storage:** Two SQLite databases on disk:
- `index.db`: chunk metadata, file hashes, last-indexed timestamps
- Vector store: LanceDB directory (binary columnar format, embeds into process)

**Embedding choice:** Local embeddings (fastembed-rs with a quantized BAAI/bge-small model, ~60MB) are recommended for v1 to avoid API cost and latency on indexing. Switch to Anthropic voyage-code-3 for better accuracy in a later phase. The pipeline should be swappable via a trait interface.

**Context recall:** Hybrid retrieval — vector similarity (semantic) + BM25 keyword (lexical) + file recency boost. Cursor research confirms hybrid outperforms pure vector for code.

---

### AI Context Manager

**Lives in:** Extension Host (TypeScript)

**Responsibilities:**
1. Receive user query + explicit @mentions
2. Call Rust Indexer for semantic search results (ranked chunks)
3. Assemble hierarchical context budget within Claude's context window
4. Inject persistent memory snippets (from conversation DB)
5. Format final `messages[]` array for the Anthropic API

**Context budget (recommended hierarchy):**

```
System prompt                    ~500 tokens  (fixed)
Current file contents           ~2,000 tokens
Directly imported/related files ~3,000 tokens
Semantically retrieved chunks   ~5,000 tokens
Conversation history (recent)   ~4,000 tokens
User's persistent memory notes  ~1,000 tokens
─────────────────────────────────────────────
Budget ceiling                  ~15,500 tokens (leaves room for response)
```

**Context window management:** When history exceeds budget, summarize older turns and persist the summary to SQLite as a "compressed memory" record. Do not silently drop — summarization preserves continuity.

---

### Auth & Credential Manager

**Lives in:** Extension Host (TypeScript)

**Two supported flows:**

**Flow 1: Anthropic OAuth (Claude Pro/Max/Teams users)**

```
1. User clicks "Login with Claude.ai"
2. Extension generates PKCE code_verifier (random 64 bytes, base64url)
   and code_challenge (SHA-256 of verifier, base64url)
3. Open system browser to:
   https://claude.ai/oauth/authorize
     ?client_id=<app_client_id>
     &response_type=code
     &code_challenge=<challenge>
     &code_challenge_method=S256
     &redirect_uri=<loopback: http://127.0.0.1:<port>/callback>
4. Local HTTP server (http module, random port) listens for redirect
5. On callback: extract code, POST to
   https://console.anthropic.com/v1/oauth/token
     { grant_type: authorization_code, code, code_verifier, client_id }
6. Store { type: "oauth", access, refresh, expires } to secure storage
7. Close local server
```

**CRITICAL POLICY NOTE (verified April 2026):** Anthropic's usage policy restricts OAuth tokens to Claude Code CLI and Claude.ai only. Third-party Electron apps using Claude.ai OAuth are not currently permitted. The safe path for v1 is **API key authentication only**. OAuth support should be deferred and tracked as a policy-dependent decision requiring Anthropic approval or a policy change.

**Flow 2: API Key (Claude Console users — recommended for v1)**

```
1. User pastes API key into settings field
2. Store to platform secure storage:
   - macOS: Keychain Services via keytar
   - Windows: Credential Manager via keytar
   - Linux: SecretService (libsecret) via keytar, fallback ~/.claude/credentials.json (mode 0600)
3. On each API request: read from keytar, inject as X-Api-Key header
4. On HTTP 401: prompt user to re-enter key
```

**Token refresh (OAuth when enabled):**
- Check `Date.now() > expires` before every request
- If expired: POST `{ grant_type: refresh_token, refresh_token }` to token endpoint
- Store new `{ access, refresh, expires }` atomically
- On refresh failure: require re-login

---

### Persistence Layer

**Technology:** SQLite via better-sqlite3 (synchronous, no callback hell) or rusqlite called from Rust indexer. Two databases, same directory (`~/.config/claudeide/<workspace-hash>/`).

**Database 1: `conversations.db`**

```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  project_path TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  title TEXT
);

CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  conversation_id TEXT REFERENCES conversations(id),
  role TEXT CHECK(role IN ('user', 'assistant')),
  content TEXT NOT NULL,         -- JSON serialized content blocks
  created_at INTEGER NOT NULL,
  token_count INTEGER,
  context_snapshot TEXT          -- JSON: which files/chunks were included
);

CREATE TABLE memory_notes (
  id TEXT PRIMARY KEY,
  project_path TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  pinned INTEGER DEFAULT 0       -- 1 = always inject into context
);

CREATE TABLE compressed_history (
  id TEXT PRIMARY KEY,
  conversation_id TEXT,
  summary TEXT NOT NULL,
  covers_message_range TEXT,     -- JSON [start_id, end_id]
  created_at INTEGER NOT NULL
);
```

**Database 2: `index.db`** (owned by Rust Indexer)

```sql
CREATE TABLE indexed_files (
  path TEXT PRIMARY KEY,
  content_hash TEXT NOT NULL,
  last_indexed INTEGER NOT NULL,
  chunk_count INTEGER
);

CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  file_path TEXT REFERENCES indexed_files(path),
  start_line INTEGER,
  end_line INTEGER,
  content TEXT,
  token_count INTEGER
);
-- Vector data lives in LanceDB sidecar directory (not in SQLite)
```

**Cursor's validated approach:** Cursor stores ALL conversation data locally in SQLite (`state.vscdb`, key-value JSON in `ItemTable`). No chat content is sent to their servers. Vector embeddings live in a separate store (Turbopuffer for Cursor, LanceDB locally for this project). This is the correct model for a privacy-respecting desktop product.

---

## Build Order

Components have clear dependency relationships. Build in this order:

```
Phase 1 ─ Foundation (no AI dependencies)
  ├── VS Code fork shell (branding, build pipeline, distribution)
  └── Persistence Layer schema + read/write utilities
        (needed by: Auth Manager, Indexer, Conversation History)

Phase 2 ─ Auth (unblocks all Anthropic API calls)
  └── Auth & Credential Manager (API key flow first; OAuth deferred)
        (depends on: Persistence Layer for token storage)

Phase 3 ─ Core AI Integration (unblocks conversation features)
  ├── Claude API Client (streaming, error handling, rate limits)
  │     (depends on: Auth Manager)
  └── Basic AI Panel (chat send/receive, no context yet)
        (depends on: Claude API Client)

Phase 4 ─ Indexer (unblocks smart context)
  └── Rust Indexer (file walk → parse → embed → store)
        (depends on: Persistence Layer for index.db)
        NOTE: Build as standalone Rust binary first, then wrap in napi-rs

Phase 5 ─ Smart Context (unblocks the core value proposition)
  └── AI Context Manager (query → retrieval → assembly → API call)
        (depends on: Rust Indexer, Persistence Layer, Claude API Client)

Phase 6 ─ Memory & History
  ├── Conversation History UI (search, browse, restore)
  └── Persistent Memory (pinned notes, compressed summaries)
        (both depend on: Persistence Layer + AI Context Manager)

Phase 7 ─ Polish & Distribution
  ├── Cross-platform packaging (Electron Builder: .dmg, .exe, .AppImage)
  ├── Auto-update mechanism
  └── Settings UI (API key management, index controls, memory management)
```

**Critical dependency:** The Rust Indexer must be built before smart context can work. But the IDE is usable (with manual @file context) before the indexer is complete. Phasing this way allows early user testing of the AI loop while indexing matures.

---

## Reference Implementations

### How Cursor Does It

**Process model:** Electron (VS Code fork) + cloud backend. Codebase indexing uses a seven-stage pipeline: Merkle tree → incremental sync → local chunking → encrypted upload → remote vector storage (Turbopuffer) → embedding cache → local retrieval. Source code never persists on Cursor's servers in plaintext.

**Storage model:** All conversation history is local SQLite (`workspaceStorage/<hash>/state.vscdb`), key-value JSON in `ItemTable`. Embeddings remote (Turbopuffer). This is confirmed by third-party reverse engineering and the existence of tools like cursor-view and cursor-db-mcp that extract local SQLite data.

**IPC:** Standard VS Code Electron IPC (ipcMain/ipcRenderer) between renderer and extension host. Cursor-specific AI features live in the extension host layer.

**Context assembly:** Hierarchical budget with current file → imports → semantic search results → history. Semantic search returns only chunk metadata (file path, line range) — actual code content stays local and is read from disk client-side before injection.

**Shadow Workspace:** A hidden background VS Code workspace that applies AI-suggested edits invisibly, runs language servers against the result, routes compiler feedback back to the model, and only surfaces the diff to the user after self-correction.

### How Windsurf Does It

**Philosophy:** "AI as a foundational layer, not an add-on." Cascade tracks the full shared timeline: files edited, terminal commands run, clipboard contents, conversation history. This timeline is the context, rather than vector search over file content.

**Key difference from Cursor:** Windsurf's Cascade is state-machine driven (tracks intent from actions, not just queries). Post-Cognition acquisition, Devin's autonomous execution engine is merged into the IDE layer.

### Where This Project Differs

| Aspect | Cursor | Windsurf | This Project |
|--------|--------|----------|--------------|
| Embeddings | Remote (Turbopuffer) | Remote | Local (LanceDB + fastembed-rs) |
| Auth | Multi-model | Multi-model | Anthropic-only (API key v1) |
| Context model | Semantic RAG | Timeline/state | Hybrid: semantic RAG + conversation memory |
| Shadow workspace | Yes | Yes (Cascade) | Deferred — Phase N feature |
| Conversation storage | Local SQLite | Likely local | Local SQLite |
| Rust usage | Yes (indexer, infra) | Unknown | napi-rs native addon in Extension Host |

**Key architectural bet:** Local embeddings + local vector store means zero per-query API cost for retrieval and no codebase data leaves the machine. This is a meaningful privacy and cost advantage over Cursor. The tradeoff is that embedding quality may be lower than Cursor's server-side voyage-code-3 models until the project migrates to better local models or the Anthropic embeddings API in a later phase.

---

## Sources

- [How Cursor Works Internally — Aditya Rohilla](https://adityarohilla.com/2025/05/08/how-cursor-works-internally/)
- [Cursor AI Deep Dive Technical Architecture — Collabnix](https://collabnix.com/cursor-ai-deep-dive-technical-architecture-advanced-features-best-practices-2025/)
- [Cursor Chat Architecture, Data Flow & Storage — DASARPAI](https://dasarpai.com/dsblog/cursor-chat-architecture-data-flow-storage/)
- [Real-world engineering challenges: building Cursor — Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/cursor)
- [How Cursor Indexes Codebases Fast — Engineer's Codex](https://read.engineerscodex.com/p/how-cursor-indexes-codebases-fast)
- [VS Code Architecture — DeepWiki](https://deepwiki.com/microsoft/vscode)
- [VS Code Sandbox Migration — VS Code Blog](https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox)
- [Electron IPC — Official Docs](https://www.electronjs.org/docs/latest/tutorial/ipc)
- [Native Code and Electron — Official Docs](https://www.electronjs.org/docs/latest/tutorial/native-code-and-electron)
- [Claude Code Authentication — Official Docs](https://code.claude.com/docs/en/authentication)
- [Token Lifecycle Management — opencode-anthropic-auth DeepWiki](https://deepwiki.com/anomalyco/opencode-anthropic-auth/3.3-token-lifecycle-management)
- [Anthropic OAuth Policy Issue — GitHub](https://github.com/AndyMik90/Aperant/issues/1871)
- [napi-rs — Official Site](https://napi.rs/)
- [Build Real-Time Codebase Indexing — CocoIndex](https://cocoindex.io/blogs/index-code-base-for-rag)
- [Build Open-Source Alternative to Cursor with Code Context — Milvus Blog](https://milvus.io/blog/build-open-source-alternative-to-cursor-with-code-context.md)
- [Cursor IDE Architecture — CursorPlus DeepWiki](https://deepwiki.com/rinadelph/CursorPlus/5.1-cursor-ide-architecture)
