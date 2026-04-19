# Research Summary — Claude Code IDE

**Synthesized:** 2026-04-19
**Source files:** STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md
**Overall confidence:** HIGH (all four files independently verified against current sources)

---

## Recommended Stack (one-liner per layer)

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Fork base** | VSCodium (not raw Code-OSS, not Monaco) | Inherits telemetry removal, marketplace patch infrastructure, and product.json branding system — duplicating this from scratch is wasted effort |
| **Packaging/distribution** | electron-builder v25.x | Cross-platform macOS/Windows/Linux artifacts from one config; differential auto-update; code signing hooks; 10x downloads vs. Electron Forge |
| **AI SDK** | @anthropic-ai/sdk v0.90.x (TypeScript) | Official SDK; native streaming; prompt caching support; Bedrock/Vertex passthrough for enterprise |
| **Credential storage** | keytar v7.x | OS-native keychain on all three platforms; same library VS Code itself uses |
| **Rust indexer transport** | stdio JSON-RPC sidecar (v1); napi-rs native addon (later) | Sidecar is simpler to develop and restart independently; migrate hot paths to native addon only when IPC overhead is measured |
| **AST parsing** | tree-sitter v0.26.x + per-language grammars | Incremental, sub-millisecond re-parse; universal language support; used by Neovim, GitHub, VS Code internally |
| **Full-text search** | tantivy v0.26.x | BM25 search; ~2x faster than Lucene; MIT-licensed; the lexical half of hybrid retrieval |
| **Local embeddings** | fastembed-rs v4/5.x (ONNX, no GPU) | Zero per-query cost; no data leaves device; AllMiniLML6V2 (384-dim) for v1; swappable behind a trait boundary |
| **File watching** | notify v7.x (Rust) | Wraps FSEvents / inotify / ReadDirectoryChangesW; drives incremental re-index on save |
| **Vector storage** | sqlite-vec v0.1.x embedded in SQLite | Zero-server, single-file; sufficient for typical codebases (<1M chunks at function granularity) |
| **Conversation/memory persistence** | SQLite via better-sqlite3 v9.x | Local, synchronous, WAL-mode concurrent reads between Rust and Node.js; same validated model Cursor uses |
| **Extension marketplace** | Open VSX (Eclipse Foundation) | Only legally unambiguous marketplace for VS Code forks; Microsoft Marketplace is explicitly banned and actively enforced |

---

## Table Stakes Features (must ship or users leave)

These are baseline expectations from anyone who has used Cursor, Windsurf, or VS Code. Absence causes immediate abandonment.

- **Syntax highlighting + LSP support** — inherited from VSCodium; do not regress
- **Integrated terminal with <50ms input latency** — Claude Code CLI runs here; Cursor's terminal lag is a documented complaint to actively avoid
- **Git integration (status, diff, staging)** — VSCodium provides; keep it working; add Claude-aware diff views
- **Inline chat with accept/reject diff UI** — any agentic file edit MUST show a diff before applying; silent modification destroys trust and is the top "I stopped using it" reason
- **Multi-file agentic edit** — Cursor Composer and Windsurf Cascade have normalized this expectation
- **Code references in chat (@file, @symbol)** — manual context targeting before smart context is ready
- **Inline ghost-text completions (opt-in default)** — absence is conspicuous; aggressive on-every-keystroke is an anti-feature (15x token waste); make it opt-in or pause-triggered
- **API key entry + keychain storage** — first-run setup screen; direct link to console.anthropic.com/keys
- **Transparent per-request token count + USD estimate** — opaque pricing is the #1 market complaint; show it always
- **Background indexing with progress indicator** — Cursor's RAM usage on large repos is a documented complaint; do not repeat it
- **Extensions from Open VSX working** — must validate that Prettier, ESLint, GitLens, and major language extensions are present before marketing

---

## Differentiating Features (why users choose this over Cursor)

These are the surfaces Cursor and Windsurf cannot offer because they are Claude Code-specific concepts. This is the reason to build this product.

**Claude Code CLI concepts made visual:**

- **Permission mode status-bar indicator** — `default / acceptEdits / plan / auto / bypassPermissions` modes shown persistently; one-click switching; no flag memorization required
- **CLAUDE.md editor with hierarchy visualization** — four-layer hierarchy (managed-policy / project / user / local) shown in a dedicated editor; warn when CLAUDE.md exceeds 200-line context limit; `/init` wizard for new projects
- **Hooks visual editor + lifecycle log** — PreToolUse, PostToolUse, SessionStart, SessionEnd, Stop, UserPromptSubmit, TeammateIdle, TaskCreated, TaskCompleted (12 events); replace raw JSON editing with a visual editor; live log shows which hooks fired, exit codes, blocked operations; Cursor and Windsurf have zero equivalent
- **MCP server registry and status panel** — install, configure, and monitor MCP servers visually; replace `claude mcp` CLI commands with a panel that shows connection status and available tools
- **Auto memory viewer (MEMORY.md sidebar)** — Claude's auto-accumulating `~/.claude/projects/<repo>/memory/MEMORY.md` exposed as an editable panel; users can audit, correct, and curate what Claude has learned; no competitor has this concept
- **Slash commands / skills palette** — `.claude/skills/` surfaced as a searchable palette (VS Code Cmd+P pattern) with descriptions, usage examples, and inline creation
- **Session browser with fork and resume** — named sessions, fork lineage, worktree links as a visual sidebar; replaces flag memorization (`--resume`, `--fork-session`)
- **Context window budget indicator** — visual token budget bar showing what's loaded (system prompt, CLAUDE.md, history, indexed files) and remaining headroom; Anthropic's own docs include this visualization; no competitor surfaces it clearly

**Infrastructure advantages:**

- **Local-first, private by default** — all embeddings, conversation history, and memory stay on-device; no codebase data leaves the machine; meaningful privacy advantage over Cursor (remote Turbopuffer) for security-conscious users and enterprises
- **Zero per-query retrieval cost** — local fastembed-rs eliminates API charges for context building; users pay only for Claude API calls, not for search
- **Cross-session conversation history search** — searchable history across all project sessions; "find the conversation where we discussed the auth redesign" is a real workflow; no competitor offers this

---

## Critical Constraints

### 1. Anthropic OAuth is banned for third-party apps — v1 ships API key only

Anthropic's usage policy (published February 19, 2026, enforced April 4, 2026) explicitly bans use of OAuth tokens from Claude Free/Pro/Max subscriptions in any third-party product. Enforcement results in account bans. This kills the original project requirement for "Anthropic OAuth login." v1 must ship with API key authentication only. OAuth against Claude Console (API billing) is a future enterprise path requiring Anthropic partnership or a policy change. Do not implement consumer OAuth under any circumstances.

### 2. Microsoft VS Code Marketplace is legally and technically off-limits

Microsoft's marketplace ToS restricts usage to VS Code, Visual Studio, GitHub Codespaces, and Azure DevOps. Forks are excluded. Enforcement is active: in April 2025, Microsoft added a runtime environment check to the C/C++ extension that hard-blocks non-Microsoft distributions. Cursor was caught reverse-proxying marketplace requests and Microsoft shut it down. Use Open VSX from day one. Validate that critical extensions are present on Open VSX before any marketing claim.

### 3. Token burn rate is a first-class product constraint, not an optimization

Anthropic acknowledged in March 2026 that Claude Code users were hitting quota limits in 19 minutes. A single "edit this file" command can consume 50,000–150,000 tokens with full context assembly. The 1M-token context window triggers a 2x price increase above 200K input tokens. Token budget management must be designed into the AI panel architecture from Phase 1, not bolted on later. Mandatory: tiered context assembly (smallest coherent context first, expand on demand), prompt caching for static contexts (CLAUDE.md, project summary), per-session spending caps, and a visible context meter. Append-only context is a product-killing anti-pattern.

### 4. Rust sidecar (stdio JSON-RPC) for v1 — not napi-rs native addon

Build the Rust indexer as a standalone binary first (easiest to develop and test), wrap it in the sidecar transport for v1, and migrate to napi-rs only when IPC latency is measured to be a problem. The sidecar is simpler to develop, independently restartable without crashing the UI, and debuggable with standard tools. Do not over-engineer the IPC bridge for v1.

### 5. Local-first storage is a non-negotiable product promise

All conversation history, embeddings, and memory stay on-device. sqlite-vec embedded in SQLite is the vector store. No external vector database service. Mandatory cloud sync of AI context is an explicitly documented anti-feature that causes enterprise and security-conscious user attrition.

---

## Architecture in One Paragraph

The IDE is a VSCodium fork (Electron shell) with custom branding and an AI Panel Extension loaded at startup via the VS Code extension host. The extension host — a separate Node.js process from the Chromium renderer — owns all AI logic: it assembles context from local SQLite databases, calls the Anthropic API with streaming, and relays tokens back to the chat UI via Electron IPC. A Rust sidecar process (v1: stdio JSON-RPC; later: napi-rs native addon) handles file watching, tree-sitter AST parsing, tantivy BM25 indexing, and fastembed-rs local embedding generation — all results stored in an `index.db` SQLite database using sqlite-vec for KNN vector search. Conversation history, compressed memory, and context snapshots live in a separate `conversations.db` SQLite file. API keys are stored in the OS keychain via keytar. Nothing leaves the machine except Anthropic API calls over HTTPS. The Chromium renderer shows the editor and the AI panel; all heavy computation happens in the extension host or the Rust sidecar, never in the renderer, preserving UI responsiveness.

---

## Top 5 Pitfalls to Avoid

### 1. Building consumer OAuth — account bans, ToS violation, project shut down
The ban is real, verified, and enforced. Any implementation path that touches Claude.ai OAuth for subscription users is a direct ToS violation with active enforcement. Ship API key auth, document it clearly, and track OAuth as a future enterprise feature requiring Anthropic partnership. Do not prototype this even experimentally with user accounts.

### 2. Touching the Microsoft Marketplace — legal exposure and active technical enforcement
Cursor's marketplace proxy was shut down publicly and caused a user trust crisis. Microsoft has the runtime check mechanism deployed and will use it on any extension they choose. Do not proxy, spoof, or mask marketplace requests. Use Open VSX. Document the limitation honestly in onboarding. Build the Open VSX extension compatibility matrix in Phase 1, not after launch.

### 3. Append-only context assembly — quota exhaustion in 19 minutes
The documented production failure mode. Design the context budget algorithm first, then the features that consume context. The architecture must treat every API call as a context budget decision: smallest coherent context, expand on model request, never full-codebase append. Prompt caching for static context must be in the initial design, not added later. Token cost per session must be measured in every sprint as a regression metric.

### 4. Minimal diff surface on the VS Code fork — rebase tax kills shipping cadence
VS Code ships 12 times per year. Every internal VS Code abstraction patched by the fork becomes a merge conflict. Keep all custom code in isolated new files and new extension points. Never patch VS Code internals. Assign one engineer as upstream watcher from day one. Automate CI builds against the latest VSCodium tag weekly.

### 5. Stale index poisoning AI responses — subtle, hard to attribute, destroys trust
A 10-minute re-index interval means agents confidently cite deleted functions and refactored APIs. File-watcher-driven incremental indexing (notify crate) must be in the indexer's initial architecture. Use a two-tier index: fast in-memory structural index updated synchronously on save; slower semantic embedding index updated asynchronously. Add an explicit index freshness indicator in the UI.

**Security (must not ship without):** Never include `.env`, `.secret*`, or credential files in any context sent to the model. Scan context before every API call. Windsurf exposed user secret keys via AI completions — this is a severe security incident that causes irreversible reputational damage.

---

## Build Order

**Phase 1 — Foundation (no AI, no networking)**
- VSCodium fork: branding, product.json, Open VSX marketplace wired, build pipeline, CI matrix (macOS/Windows/Linux)
- Code signing + notarization pipeline in CI — treat as infrastructure, not a launch task
- SQLite persistence layer: schema for `conversations.db` and `index.db`, read/write utilities
- Startup time baseline established and enforced in CI (<1.5s to interactive on M1)
- Open VSX extension compatibility matrix documented

**Phase 2 — Auth + Core AI Loop**
- API key entry, keychain storage via keytar, first-run setup screen
- Anthropic API client: streaming, error handling, rate limit handling, prompt caching
- Basic AI chat panel: send/receive, streaming response display
- Context budget algorithm: tiered assembly, not append-only
- Token count + USD cost display (per-message)
- Per-session spending cap UI

**Phase 3 — Claude Code CLI Integration (the differentiators)**
- Permission mode status bar indicator + switcher
- CLAUDE.md editor with hierarchy visualization and `/init` wizard
- Integrated terminal with <50ms latency target enforced
- Inline accept/reject diff UI for agentic edits
- MCP server panel (list, configure, status)
- Slash commands / skills palette

**Phase 4 — Codebase Indexer**
- Rust sidecar (stdio JSON-RPC): file walker, tree-sitter parsing, tantivy BM25, fastembed-rs embeddings, sqlite-vec storage, notify file watcher
- Incremental re-index on file save (not polling)
- Two-tier index: fast structural + async semantic
- Index freshness UI indicator
- Linux inotify limit detection and user warning
- Windows long path support

**Phase 5 — Smart Context + Memory**
- Context Manager: hybrid retrieval (BM25 + vector + recency), context budget assembly
- Semantic codebase search UI (natural language queries)
- Hooks visual editor + lifecycle log
- Auto memory viewer (MEMORY.md sidebar panel)
- Cross-session conversation history search
- Session browser with fork/resume and worktree launcher
- Context window budget indicator

**Phase 6 — Team + Distribution**
- Team skills library (project vs. personal vs. installed)
- Org-managed CLAUDE.md (read-only managed-policy layer)
- Cross-platform distribution: auto-update, delta packages, Squirrel.Windows first-launch delay
- Agent Teams dashboard (only after `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` stabilizes)

Rationale for this order: Auth gates every Anthropic API call. The basic AI loop must exist before any Claude Code-specific surface is built. The indexer is the hardest engineering problem and should mature while the Claude Code CLI features (which don't depend on it) are being built in Phase 3. Smart context depends on the indexer. Distribution is validated iteratively with signed alpha builds from Phase 1.

---

## Open Questions

1. **Sidecar vs. napi-rs timing** — STACK.md says sidecar first; ARCHITECTURE.md prefers napi-rs from the start. Resolution: build Rust indexer as standalone binary first, wrap in sidecar transport for v1, migrate to napi-rs only when IPC latency is measured. The Phase 4 research spike should benchmark this explicitly.

2. **Embedding model selection** — fastembed-rs supports AllMiniLML6V2 (384-dim, faster), BGESmallENV15 (384-dim, higher quality), BGEBaseENV15 (768-dim, slower). Needs benchmarking on representative codebases (100K+ LOC) for CPU latency vs. retrieval quality. Flag for Phase 4 research.

3. **Context selection algorithm weighting** — Hybrid BM25 + vector + recency is the recommended architecture, but the weighting is both a tech and product decision. Research Cursor's and Windsurf's specific approaches in the phase that builds smart context (Phase 5).

4. **CLI subprocess vs. direct API for agentic tasks** — For conversation, direct `@anthropic-ai/sdk` gives full context window control. For agentic file edits and bash execution, the Claude Code CLI handles the tool use loop, permission prompts, and safety checks. Likely resolution: direct API for conversation, CLI subprocess for agentic execution. Decide in Phase 3 when the integrated panel is designed.

5. **sqlite-vec performance ceiling at scale** — At 1M vectors at 3072 dimensions, queries take ~8s (unacceptable). At 384 dimensions and typical per-developer codebases, latency should be acceptable — but needs empirical validation in Phase 4 against large monorepos.

6. **Open VSX extension coverage audit** — Needs a formal compatibility check before any marketing claim. Specifically: Prettier, ESLint, GitLens, Python (Pyright as Pylance alternative), C/C++ (clangd), Rust Analyzer, Go, Java, Docker. Missing critical extensions need a documented mitigation plan.

7. **Enterprise OAuth path** — Claude Console OAuth (API billing, not consumer subscription) may be permitted. Clarify with Anthropic whether a third-party Electron app can implement Console OAuth for enterprise customers, and what approval process is required. Affects Phase 2 auth design.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Fork base (VSCodium) | MEDIUM-HIGH | Build process verified; patch maintenance overhead is real and must be budgeted |
| Auth (API key only for v1) | HIGH | OAuth ban verified against official Anthropic policy (Feb 2026); no ambiguity |
| Marketplace (Open VSX only) | HIGH | Microsoft enforcement confirmed via multiple news sources and active C/C++ extension block |
| Rust indexer stack (tree-sitter, tantivy, notify) | HIGH | Official crates, production-verified in multiple 2025 open-source projects |
| Local embeddings (fastembed-rs) | MEDIUM | Crate confirmed active; embedding quality/latency on CPU needs project-specific benchmarking |
| sqlite-vec for vector storage | MEDIUM-HIGH | v0.1.x stable; performance ceiling documented; typical codebase sizes should be fine but needs validation |
| Token burn rate as critical constraint | HIGH | Verified by Anthropic's own public acknowledgment (March 2026) |
| Differentiator features (CLAUDE.md, hooks, MCP, memory) | HIGH | All verified against Claude Code CLI docs; no competitor has these surfaces |
| electron-builder for distribution | HIGH | Industry standard; cross-platform CI confirmed |

Overall research confidence: HIGH. The main unknowns are empirical (embedding model selection, sqlite-vec at scale, IPC latency measurement) — resolvable with targeted benchmarks in their respective phases.

---

## Sources (aggregated)

- VSCodium build system: https://deepwiki.com/VSCodium/vscodium
- Cursor architecture: https://deepwiki.com/rinadelph/CursorPlus/5.1-cursor-ide-architecture
- Anthropic OAuth ban (Feb 2026): https://www.theregister.com/2026/02/20/anthropic_clarifies_ban_third_party_claude_access/
- Claude Code auth docs: https://code.claude.com/docs/en/authentication
- Anthropic Claude Code quota crisis (March 2026): https://www.theregister.com/2026/03/31/anthropic_claude_code_limits/
- Microsoft C/C++ extension enforcement (April 2025): https://www.theregister.com/2025/04/24/microsoft_vs_code_subtracts_cc_extension/
- tree-sitter crate: https://crates.io/crates/tree-sitter (v0.26.3)
- tantivy crate: https://docs.rs/crate/tantivy/latest (v0.26.0)
- fastembed-rs: https://github.com/Anush008/fastembed-rs
- napi-rs v3: https://napi.rs/blog/announce-v3
- @anthropic-ai/sdk TypeScript: https://github.com/anthropics/anthropic-sdk-typescript (v0.90.x)
- sqlite-vec: https://alexgarcia.xyz/blog/2024/sqlite-vec-stable-release/index.html
- electron-builder: https://www.electron.build/
- VS Code architecture: https://deepwiki.com/microsoft/vscode
- Augment Code — To Fork or Not to Fork: https://www.augmentcode.com/blog/to-fork-or-not-to-fork
