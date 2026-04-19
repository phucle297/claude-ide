# Claude Code IDE

VS Code fork (VSCodium base) built exclusively for Claude Code power users. Stack: TypeScript (Electron + Extension Host), Rust native addon (napi-rs), SQLite, @anthropic-ai/sdk.

## Architecture

| Component | Runtime | Responsibility |
|-----------|---------|----------------|
| Fork shell | Electron (Chromium + Node.js) | Editor UI, tabs, layout, settings |
| AI panel | Extension Host (TypeScript) | Chat UI, context picker, diff viewer, accept/reject |
| Rust indexer | napi-rs native `.node` addon | File watching, tree-sitter AST, tantivy BM25, fastembed embeddings, sqlite-vec |
| Auth manager | Extension Host (TypeScript) | Anthropic OAuth PKCE, API key fallback, keytar keychain |
| Persistence | SQLite (WAL mode) | Conversation history, project memory, index metadata |
| Claude client | Extension Host (TypeScript) | Context assembly, Anthropic API streaming, prompt caching |

**IPC path:** Chromium renderer → Electron IPC → Extension Host → napi-rs sync/async → Rust → SQLite

## Branching from VSCodium

- Branding lives in `product.json` (app name, icons, marketplace URL, telemetry endpoints)
- Custom patches go in `patches/user/*.patch` — keep them minimal
- Do NOT patch deep VS Code internals (workbench layout, extension host internals) — upstream syncs conflict
- VSCodium syncs monthly; track their upstream merge PRs to time our sync windows

## Non-Negotiable Rules

- **Never proxy Microsoft Marketplace** — ToS violation, actively enforced; use Open VSX Registry only
- **Rust indexer = napi-rs addon, not sidecar** — avoids 2-4ms IPC overhead per call
- **SQLite in WAL mode** — concurrent reads between Rust (rusqlite) and Node.js (better-sqlite3)
- **All agentic edits show diff UI before applying** — silent modifications destroy user trust (top abandonment reason)
- **Show token count + USD estimate on every request** — opaque pricing is the #1 market complaint in 2025

## Key Libraries

```
TypeScript:  @anthropic-ai/sdk ^0.90, better-sqlite3 ^9, electron-builder ^25, keytar ^7
Rust:        tree-sitter 0.26, tantivy 0.26, fastembed 4/5 (ONNX), notify 7, rusqlite, sqlite-vec 0.1, napi-rs
```

## Build Commands

(Populate once VSCodium fork is cloned and build system is wired up)

```bash
# VSCodium build flow (once set up):
# npm run build-vscodium     — prepare + patch + compile
# npm run package            — electron-builder artifacts
# cargo build --release      — Rust indexer (napi-rs produces .node file)
```

## Extension API Contract

The supported customization surface is the VS Code extension API — treat it as the ABI boundary. Avoid reaching into VS Code internals (`src/vs/workbench/...`) except for the minimum required for the AI panel integration. Every internal touch is a potential upstream conflict.

## Prompt Caching

Always use prompt caching for large context assemblies (system prompt + codebase chunks). Cache breakpoints on the system prompt and the static context prefix. See @anthropic-ai/sdk docs for `cache_control` blocks.
