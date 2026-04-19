# Stack Research — Claude Code IDE

**Researched:** 2026-04-19
**Overall confidence:** MEDIUM-HIGH (all recommendations verified against current sources)

---

## Recommended Stack

### Fork Base

**Recommendation: VSCodium as the fork base, not raw VS Code or Monaco**

VSCodium is the right starting point. It is not a separate product — it is scripts that build Microsoft's own MIT-licensed `microsoft/vscode` source code into binaries, stripping telemetry and Microsoft-specific licensing. The output is functionally identical to VS Code with the same extension API surface.

Why VSCodium over alternatives:

- **vs. raw `microsoft/vscode`**: You still need to strip telemetry and Microsoft marketplace references anyway. VSCodium's patch system and `product.json` customization infrastructure does this for you and documents how to maintain it as upstream merges happen. Doing it yourself from scratch duplicates their work poorly.
- **vs. Monaco Editor alone**: Monaco is only the text-editor widget extracted from VS Code for browser embedding. It has no file tree, no extension host, no debugger, no terminal, no language server orchestration. You would be rebuilding VS Code from scratch around Monaco. This is how Theia works and it took years to reach parity — wrong path for a v1.
- **vs. Tauri**: Tauri is not a VS Code fork. It is a general desktop app framework. Using Tauri would mean writing the entire IDE surface (file tree, tabs, terminal, diff viewer, extension host) from scratch. The value proposition of forking VS Code is inheriting all of that for free.

**How to fork VSCodium:** Clone the VSCodium repo. The build system has three phases: source preparation, patching (via `patches/*.patch`), and compilation. Custom branding lives in `product.json` — app name, icons, extension marketplace URL, telemetry endpoints. User-specific patches go in `patches/user/*.patch`. Environment variables in `utils.sh` control all branding placeholders. The VSCodium build scripts produce macOS `.dmg`, Windows installer (NSIS/MSI), and Linux (AppImage, `.deb`, `.rpm`) artifacts from a single codebase. This is the same pattern Cursor and Windsurf use, confirmed by their architecture analysis.

**Extension marketplace:** VSCodium defaults to Open VSX Registry (Eclipse Foundation). For a focused Claude-only IDE, this is acceptable — language servers, formatters, linters, themes, and debuggers are all present. Microsoft's proprietary extensions (Copilot, some first-party language tools) are not available, but those are competitors or irrelevant here.

**Confidence:** MEDIUM-HIGH. VSCodium's build process is well-documented. The fork-based approach is confirmed by Cursor and Windsurf's architecture. The Open VSX extension gap is a known and manageable constraint.

---

### Rust Layer

The Rust layer runs as a sidecar process (spawned by the Electron main process) or as a native Node addon via napi-rs. Use the sidecar model first — it is simpler to develop and debug, and allows independent restart without crashing the UI process. Migrate hot-path components to a native addon only if IPC overhead becomes measurable.

#### Parsing and AST

| Crate | Version | Purpose |
|-------|---------|---------|
| `tree-sitter` | 0.26.x | Incremental AST parsing, language-agnostic |
| `tree-sitter-rust`, `tree-sitter-typescript`, `tree-sitter-python`, etc. | 0.24.x / 0.26.x | Per-language grammars |

tree-sitter is the only viable choice for a multi-language codebase index. It is incrementally updated (sub-millisecond re-parse on file edits), produces concrete syntax trees that can be queried with tree-sitter query syntax to extract functions, classes, and symbol definitions, and has grammars for every mainstream language. It is the parser used by Neovim's syntax highlighting, GitHub's code navigation, and is the same library VS Code's built-in syntax support uses internally.

Do not use `syn` (Rust-only) or language-specific parsers. The goal is a universal indexer that works across whatever languages are in the user's repo.

#### Full-Text and Hybrid Search

| Crate | Version | Purpose |
|-------|---------|---------|
| `tantivy` | 0.26.x | BM25 full-text search index |

Tantivy is Lucene for Rust — roughly 2x faster than Lucene in benchmarks, MIT-licensed, actively maintained. It handles the keyword-search half of hybrid retrieval. The code search pattern that works in production (verified in multiple 2025 open-source projects like code-sage and opencode-codebase-index) is: tree-sitter extracts semantic chunks (function bodies, class definitions), tantivy indexes them for BM25 retrieval, vector embeddings handle semantic similarity, and Reciprocal Rank Fusion (RRF) merges the two result sets.

#### Local Embeddings

| Crate | Version | Purpose |
|-------|---------|---------|
| `fastembed` | 4.x / 5.x | Local ONNX-based text embeddings, no GPU required |

fastembed-rs runs ONNX models via `ort` (the Rust binding to ONNX Runtime). It supports quantized models (e.g., `BGESmallENV15Q`) that run acceptably on laptop CPUs. No GPU needed, no external service, no network call. This is critical for the zero-cost-to-builder model — embeddings must be generated locally so users do not need another paid API.

Model recommendation: `AllMiniLML6V2` (384-dim) for initial implementation, upgradeable to `BGESmallENV15` or `BGEBaseENV15` as hardware budgets allow. Keep it swappable behind a trait boundary.

#### File Watching

| Crate | Version | Purpose |
|-------|---------|---------|
| `notify` | 7.x | Cross-platform file system events |

`notify` wraps OS-native watchers (FSEvents on macOS, inotify on Linux, ReadDirectoryChangesW on Windows). Use it to drive incremental re-indexing on file saves rather than polling. Well-maintained, widely used in the Rust ecosystem (used by cargo, watchexec, etc.).

#### IPC Transport (sidecar mode)

| Crate | Version | Purpose |
|-------|---------|---------|
| `tokio` | 1.x | Async runtime for the sidecar |
| `serde` + `serde_json` | 1.x | Message serialization |

Communicate over stdio JSON-RPC (newline-delimited JSON frames). This is the same pattern used by Language Server Protocol — it works, it is debuggable with standard tools, and it does not require native addon compilation on every platform. The TypeScript side uses a simple readline-based reader.

**Confidence:** HIGH for tree-sitter, tantivy, notify. MEDIUM for fastembed (embedding quality and latency on CPU needs profiling — flag for Phase-specific research).

---

### TypeScript / UI Layer

The UI layer is the forked VS Code codebase itself (TypeScript) plus an extension host extension that adds the Claude panel. This is how Cursor works: the core AI features live in a custom extension loaded at IDE startup, not in the VS Code core layer. Changes to the core are limited to: branding, custom activity bar icons, and injecting the extension at startup.

| Package | Version | Purpose |
|---------|---------|---------|
| `@anthropic-ai/sdk` | 0.90.x | Anthropic API client |
| `@napi-rs/cli` | 3.x | Build tooling for Rust native addons |
| `napi` (Rust crate) | 3.x | Rust-side napi-rs binding |
| `better-sqlite3` | 9.x | SQLite access from Node.js side |
| `keytar` | 7.x | OS keychain storage for API keys and tokens |

The extension panel implements: conversation UI (React, already available in the VS Code webview model), Claude API calls via `@anthropic-ai/sdk`, and IPC calls to the Rust sidecar for index queries. VS Code webviews run in sandboxed Electron renderers and communicate with the extension host via `vscode.postMessage` — keep this boundary clean.

Use `keytar` (the same library VS Code itself uses) for credential storage. On macOS it writes to Keychain, on Linux to Secret Service / libsecret, on Windows to the Credential Manager. Do not store API keys in plaintext config files.

**Confidence:** HIGH. This layer is exactly the VS Code extension model — well-documented, proven by all AI IDE forks.

---

### AI Integration

#### SDK

Use `@anthropic-ai/sdk` v0.90.x (current as of April 2026). Initialize with the API key from the user's stored credential:

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  apiKey: await keytar.getPassword('claude-code-ide', 'api-key'),
});
```

The SDK handles streaming natively via `client.messages.stream()`. Streaming is required for the chat panel — do not use the non-streaming endpoint for interactive use.

#### Authentication Strategy

**CRITICAL FINDING: Anthropic has explicitly banned third-party use of OAuth subscription tokens as of February 2026.**

The official Anthropic policy, published February 19, 2026, states: "Using OAuth tokens obtained through Claude Free, Pro, or Max accounts in any other product, tool, or service is not permitted and constitutes a violation of the Consumer Terms of Service." Enforcement of this ban on third-party harnesses began April 4, 2026. Any attempt to reimplement the OAuth PKCE flow against Anthropic's consumer endpoints will result in enforcement action and potential account bans for users.

This means the original project requirement "Anthropic OAuth login (users connect their existing Anthropic subscription)" is **not buildable** under the current terms.

**What is permitted and works:**

1. **Direct API key (primary path):** Users create an API key at `console.anthropic.com` and paste it into the IDE. This is the standard BYOK pattern supported by Cursor, Windsurf, JetBrains AI, and every other third-party tool. The `@anthropic-ai/sdk` supports this natively. Store the key with `keytar`.

2. **Claude Console OAuth (enterprise path):** Organizations using Claude Console (API billing) can authenticate their IDE against the Console. This is different from the consumer subscription OAuth — it is API-based, not subscription-based.

3. **Bedrock / Vertex (enterprise-cloud path):** Support AWS Bedrock or Google Vertex AI as auth backends for organizations that route through those. The SDK has `AnthropicBedrock` and `AnthropicVertex` clients.

The UI should have a first-run setup screen: "Enter your Anthropic API key" with a direct link to `console.anthropic.com/keys`. This is how Cursor and JetBrains handle it. Users who want subscription-priced access must use the official Claude Code CLI or Claude.ai — this is Anthropic's explicit product boundary.

**Confidence:** HIGH. Verified against official Anthropic policy documentation and Claude Code authentication docs.

---

### Local Storage / Embeddings

#### Vector Storage

**Recommendation: `sqlite-vec` (v0.1.x) embedded in the same SQLite database as conversation history**

sqlite-vec is a SQLite extension (pure C, no dependencies) that adds KNN vector search. It runs anywhere SQLite runs — macOS, Windows, Linux, ARM. For the codebase index use case, the performance ceiling is meaningful but acceptable: at 1M vectors it slows, but a typical developer codebase (even a large monorepo) indexes to well under 1M chunks at function/class granularity.

The single-file, zero-setup nature is the decisive advantage. Users do not run a server. There is no port conflict, no daemon to manage, no install step. The IDE opens and the index is in `~/.config/claude-code-ide/projects/<project-hash>/index.db`.

Do not use Qdrant (even local mode) for v1. Qdrant requires running a separate process, adds deployment complexity, and is sized for workloads far beyond what a per-developer local index needs. Qdrant is the right answer if you build a shared team index server — that is a future milestone.

Rust access: use the `sqlite-vec` crate (0.1.x) which statically links the C extension. TypeScript access: use `better-sqlite3` with the sqlite-vec extension loaded.

#### Conversation and Memory Persistence

Use SQLite via `better-sqlite3` (Node.js side) for:
- Conversation history per project (messages, timestamps, model used)
- Persistent memory snippets (user-tagged or AI-extracted facts about the codebase)
- Index metadata (file hashes for incremental re-index detection)

Schema lives in the same `index.db` file. SQLite WAL mode handles concurrent reads from the Rust sidecar and the Node.js extension process.

**Confidence:** HIGH for sqlite-vec architecture. MEDIUM for sqlite-vec performance at scale — needs benchmarking against representative large codebases in a dedicated phase.

---

### Build and Distribution

**Recommendation: electron-builder v25.x for packaging, GitHub Actions for CI**

electron-builder is the correct choice over Electron Forge for this project:

- **Cross-platform from one repo:** electron-builder supports building macOS `.dmg` + `.zip`, Windows NSIS installer + MSI, and Linux AppImage + `.deb` + `.rpm` from the same configuration. Electron Forge requires running on the target platform for each build — incompatible with a simultaneous three-platform v1 requirement.
- **Auto-update built in:** electron-builder includes `electron-updater` for differential auto-update. Users get notified of new versions and can update without a full reinstall. This is table stakes for a desktop IDE.
- **Code signing support:** Handles macOS notarization and Windows Authenticode signing in CI.
- **Track record:** electron-builder has 10x the weekly downloads of Electron Forge and is used by the production apps this project is modeled on.

**CI matrix:** Use GitHub Actions with `macos-latest`, `windows-latest`, and `ubuntu-latest` runners in a build matrix. Use `electron-builder`'s Docker image for Linux builds. Sign macOS via Apple Developer certificate stored as a GitHub Actions secret.

**Rust sidecar build:** The Rust binary compiles separately via `cargo build --release --target <triple>` in the same CI pipeline. The electron-builder `extraResources` config bundles the Rust binary into the app package. The Electron main process spawns it on startup via `child_process.spawn`.

For native addons (if you migrate hot paths from sidecar to napi-rs later), `@napi-rs/cli` handles the cross-compilation targets and produces pre-built `.node` files per platform/arch that electron-builder bundles automatically.

**Confidence:** HIGH. electron-builder is the established tool for this exact use case.

---

## What NOT to Use

| Technology | Why Not |
|------------|---------|
| **Tauri** | Not a VS Code fork base. Using Tauri means writing the entire IDE surface from scratch. Only viable for a greenfield IDE that does not need VS Code extension compatibility. Wrong tradeoff for v1. |
| **Monaco Editor alone** | Browser editor widget only. No file tree, terminal, debugger, extension host. Would require rebuilding everything VS Code already provides. |
| **Eclipse Theia** | Another VS Code-compatible IDE framework but adds its own abstraction layer, has slower upstream VS Code merges, and a smaller community. VSCodium is simpler for a fork. |
| **Qdrant (local)** | Requires a running server process. Adds operational complexity to a local desktop tool. Correct for a team server feature (future), wrong for per-developer local index. |
| **Chroma** | Python-native, Node.js bindings are secondary. Wrong language for the stack. |
| **Consumer OAuth PKCE for subscriptions** | Explicitly banned by Anthropic as of February 2026. Using it violates Terms of Service and results in account bans. Do not implement. |
| **Electron Forge** | Build-on-target-only model conflicts with a simultaneous three-platform v1. |
| **Raw `microsoft/vscode` fork** | Requires manually reimplementing everything VSCodium's patch infrastructure already does (telemetry removal, marketplace replacement, license cleanup). Duplication of effort. |
| **WASM for Rust indexer** | WASM sandboxing prevents direct file system access. The indexer needs to read the entire project tree. Use a native sidecar or native addon instead. |
| **LanceDB** | Newer, less battle-tested than sqlite-vec for the embedded no-server use case. sqlite-vec has a simpler operational model. |

---

## Confidence Levels

| Area | Confidence | Basis |
|------|------------|-------|
| Fork base (VSCodium) | MEDIUM-HIGH | VSCodium build docs confirmed, Cursor/Windsurf architecture verified via multiple sources |
| tree-sitter for parsing | HIGH | Official crate, version confirmed, widely used in production 2025 projects |
| tantivy for BM25 | HIGH | Official crate (v0.26.x confirmed), well-established |
| fastembed for local embeddings | MEDIUM | Crate confirmed active, but embedding quality/latency on CPU requires project-specific benchmarking |
| napi-rs for Rust/Node bridge | HIGH | v3 confirmed, Electron compatibility verified |
| @anthropic-ai/sdk | HIGH | Official SDK, v0.90.x confirmed, Context7 docs verified |
| OAuth ban (no subscription OAuth) | HIGH | Verified against official Anthropic policy (Feb 2026) and Claude Code auth docs |
| sqlite-vec for vector storage | MEDIUM-HIGH | v0.1.x confirmed stable, performance ceiling at 1M vectors documented, sufficient for typical codebases |
| electron-builder for distribution | HIGH | Industry standard, widely deployed, cross-platform capability confirmed |

---

## Open Questions

1. **Embedding model selection:** fastembed-rs supports multiple ONNX models. `AllMiniLML6V2` (384-dim) is lightweight; `BGESmallENV15` (384-dim, higher quality); `BGEBaseENV15` (768-dim, slower). Needs benchmarking on representative codebases (100K+ LOC) to find the CPU latency / quality tradeoff that is acceptable at initial index time. Flag for Phase 2 (indexing phase) research.

2. **Context selection algorithm:** "Smart context inclusion" — how does the IDE decide which files to send to Claude when the user asks a question? The tree-sitter + tantivy + vector hybrid index enables this, but the ranking/selection algorithm is a product decision, not just a tech decision. Research Cursor's and Windsurf's approaches in the phase that builds this feature.

3. **Claude Code CLI wrapping vs. API directly:** PROJECT.md defers the decision on whether to wrap the `claude` CLI or call the Anthropic API directly. For the conversation panel, direct API (`@anthropic-ai/sdk`) gives full control over context window assembly. For executing agentic tasks (file edits, bash commands), the Claude Code CLI may be preferable as it handles tool use, permissions, and the full agentic loop. This decision should be made in the phase that implements the integrated panel — likely a hybrid: direct API for conversation, CLI subprocess for agentic execution.

4. **Index size limits:** At what codebase size does sqlite-vec become too slow for real-time context queries? Needs measurement. The code-sage project (2025) reports that at 1M vectors / 3072 dimensions, queries take ~8s — unacceptable. At 384 dimensions (fastembed AllMiniLML6V2) and typical codebase sizes (under 200K chunks for most repos), latency should be acceptable. Validate this assumption in the indexing phase.

5. **Open VSX gap for users:** If users expect a specific VS Code extension that is not on Open VSX, they will be blocked. Assess whether the most common power-user extensions (Prettier, ESLint, GitLens, language extensions) are available on Open VSX before shipping. Initial research suggests they are, but validate this in Phase 1.

---

## Sources

- VSCodium build system: https://deepwiki.com/VSCodium/vscodium
- Cursor architecture: https://deepwiki.com/rinadelph/CursorPlus/5.1-cursor-ide-architecture
- Windsurf indexing: https://windsurf.com/security
- tree-sitter crate: https://crates.io/crates/tree-sitter (v0.26.3)
- tantivy crate: https://docs.rs/crate/tantivy/latest (v0.26.0)
- fastembed-rs: https://github.com/Anush008/fastembed-rs
- napi-rs v3: https://napi.rs/blog/announce-v3
- Anthropic SDK TypeScript: https://github.com/anthropics/anthropic-sdk-typescript (v0.90.x)
- Anthropic OAuth ban: https://www.theregister.com/2026/02/20/anthropic_clarifies_ban_third_party_claude_access/
- Claude Code auth docs: https://code.claude.com/docs/en/authentication
- sqlite-vec: https://alexgarcia.xyz/blog/2024/sqlite-vec-stable-release/index.html
- electron-builder: https://www.electron.build/
- Electron v40/41 releases: https://releases.electronjs.org/
