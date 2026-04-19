# Pitfalls Research — Claude Code IDE

**Domain:** AI-powered VS Code fork (Rust + TypeScript, Claude-only)
**Researched:** 2026-04-19
**Overall confidence:** HIGH (verified across multiple official and industry sources)

---

## Critical Pitfalls
*(These will kill the project if not addressed early)*

---

### 1. Microsoft Marketplace Lockout

**What goes wrong:** The Visual Studio Marketplace Terms of Service restrict usage to "in-scope products" — Visual Studio, VS Code, GitHub Codespaces, Azure DevOps. Forks are explicitly excluded. Microsoft enforces this not just contractually but technically: in April 2025 they added a runtime environment check to the C/C++ extension (v1.24.5) that hard-blocks the extension on any non-Microsoft VS Code distribution. Cursor had been using a reverse proxy to mask marketplace requests and Microsoft shut it down. This pattern will escalate — every high-value Microsoft extension (Python/Pylance, C#, Docker, Jupyter) has the same enforcement mechanism available.

**Warning signs:**
- Users report that Microsoft-published extensions fail silently or show "this extension is not available for this product"
- Extensions install but fail on activation with environment-check errors
- Microsoft releases a new extension version mid-sprint with a runtime check

**Prevention:**
- Use Open VSX Registry (Eclipse Foundation) as the primary extension marketplace — it is the only legally unambiguous option for forks
- Document clearly in onboarding that Microsoft-marketplace extensions are not supported
- Maintain a curated list of Open VSX alternatives for the most commonly needed extensions (clangd for C/C++, Pylance alternative via Pyright, etc.)
- Do not attempt to proxy or spoof Microsoft marketplace endpoints — this is a ToS violation and active enforcement risk

**Phase:** Phase 1 (Fork Setup). This decision gates the entire distribution model. Decide before a single user-facing screen is built.

---

### 2. VS Code Upstream Drift and Rebase Tax

**What goes wrong:** VS Code ships monthly releases (12/year). Each release touches thousands of files. The longer a fork diverges without syncing, the larger the rebase becomes — and the more custom patches conflict with upstream changes. Teams consistently underestimate the ongoing labor cost: at scale (Cursor, Windsurf), this is a multi-engineer effort every month. Features users expect from VS Code (new settings sync, accessibility improvements, language server updates) arrive weeks late or not at all. Worse, VS Code's architecture is intentionally not designed to be forked cleanly — Microsoft has stated the extension API is the supported surface area, not the source.

**Warning signs:**
- Custom patches are touching deep internal VS Code abstractions (workbench layout, extension host, renderer process)
- A single upstream PR changes a file your fork has also modified
- Monthly release lag grows from 1 week to 3 weeks to "we skipped this version"

**Prevention:**
- Base the fork on VSCodium (not raw Code-OSS) to inherit their patch-management infrastructure (`prepare_vscode.sh` pattern)
- Minimize the diff surface: keep custom code in isolated new files and new extension points rather than patching existing VS Code internals
- Assign one engineer as "upstream watcher" — their job each release cycle is to pull upstream, resolve conflicts, and validate that the fork still builds and passes smoke tests
- Automate CI to build against the latest VSCodium tag weekly and alert on build failures before they accumulate

**Phase:** Phase 1 (Fork Setup) and ongoing. Establish the sync cadence and tooling in Phase 1; do not defer.

---

### 3. Claude API Token Burn Rate — Quota Exhaustion in Production

**What goes wrong:** In March 2026, Anthropic acknowledged that Claude Code users were hitting quota limits in 19 minutes instead of the expected 5 hours — "the top priority for the team." The root cause: a single "edit this file" command can consume 50,000–150,000 tokens when full context is assembled across multi-turn conversation, growing context windows, and tool-use round trips. An IDE that naively passes the full indexed codebase plus conversation history on every request will exhaust even Max-plan users in a single work session. The 1M token context window on Sonnet 4.6 triggers a 2× price increase above 200K input tokens, turning innocent-looking "send full context" implementations into dollar-per-conversation charges.

**Warning signs:**
- Integration tests that measure token counts per interaction show >50K tokens on simple queries
- First internal testers report hitting rate limits within an hour of use
- API cost per active user/day exceeds $1 in testing (unsustainable at scale)

**Prevention:**
- Implement tiered context assembly: start with the smallest coherent context (current file + immediate imports), expand only when the model requests more
- Cache prompt prefixes using Anthropic's prompt caching API — large static contexts (project summary, CLAUDE.md) can be cached and reused across turns at ~10% of full token cost
- Set hard per-session token budgets that users can see and adjust, surfaced in the UI
- Use conversation compaction: after N turns, summarize the conversation into a denser form before continuing
- Track tokens-per-feature during development; treat token efficiency as a first-class engineering metric from Phase 1

**Phase:** Phase 2 (AI Panel integration) must design for this. Phase 3 (indexing) must design context-sending limits. Do not leave for optimization later.

---

### 4. Codebase Index Staleness and Context Rot

**What goes wrong:** A code index is a snapshot. The moment a file is saved, the index is wrong. Cursor's re-index interval was 10 minutes — meaning agents confidently reference deleted functions, phantom APIs, and refactored class names. At 100K+ file codebases, embedding the entire repo takes substantial time and memory (embeddings can reach 10 GB for large repos). "Context rot" compounds: search results are loaded directly into the model's context window, and stale results degrade response quality in ways that are subtle — the model doesn't announce it's confused, it just produces slightly wrong answers that are hard to attribute to the index.

**Warning signs:**
- Test queries against a recently-refactored file return the old function signature
- Indexing a medium-sized monorepo (50K files) takes >60 seconds on first open
- Memory usage climbs steadily during an IDE session without plateauing
- Users report AI ignoring a file they just edited

**Prevention:**
- Implement file-watcher-driven incremental indexing (Rust + `notify` crate): re-index only changed files within milliseconds of save, not on a polling interval
- The Rust indexer should maintain a dirty set — files that have changed since last embed. Prioritize dirty files in the embedding queue
- Use a two-tier index: fast in-memory structural index (AST, symbol table) updated synchronously on save; slower semantic embedding index updated asynchronously
- Scope the initial index to the files the user actually has open and their immediate dependencies; full repo index builds in the background
- Add explicit "index freshness" indicators in the UI so users understand when context may be stale

**Phase:** Phase 3 (Codebase Indexing). This is the hardest engineering problem in the project. Reserve a full phase for it.

---

### 5. OAuth PKCE Deep Link Interception (Security)

**What goes wrong:** Electron apps are public clients — the binary can be decompiled and a client secret extracted, so PKCE (Proof Key for Code Exchange) is required. The failure mode is the redirect URI. If the app registers a custom URI scheme (`claudeide://oauth/callback`) at the OS level, any other app on the machine can register the same scheme and intercept the auth code before the IDE receives it. On macOS and Windows, custom URI scheme registration is first-come-first-served; a malicious app installed before the IDE wins the race. This is a known attack vector for desktop OAuth (RFC 8252, Section 8.1).

**Warning signs:**
- Auth callback lands in a browser tab instead of the app
- Tests show that registering the same custom URI scheme from a second process succeeds
- The Anthropic OAuth redirect URI allows wildcard or http://localhost patterns

**Prevention:**
- Use `http://127.0.0.1` loopback redirect with a randomly assigned ephemeral port (RFC 8252 Section 7.3) instead of custom URI schemes — only the process that bound the port can receive the response. This is more secure than custom protocols on all platforms
- Implement PKCE correctly: generate `code_verifier` (cryptographically random, 43–128 chars), derive `code_challenge` as BASE64URL(SHA256(code_verifier)), send challenge to Anthropic, receive auth code, exchange code + verifier for tokens
- Store tokens using Electron's `safeStorage` API (wraps OS keychain: Keychain on macOS, Credential Manager on Windows, libsecret on Linux) — never store tokens in `localStorage` or plaintext files
- Validate the `state` parameter on every callback to prevent CSRF

**Phase:** Phase 2 (Auth). Must be correct on first implementation. Security retrofits are expensive.

---

## Common Pitfalls
*(These cause significant rework if ignored)*

---

### 6. Rust ↔ Electron IPC Performance Degradation

**What goes wrong:** The Rust indexer communicates with the TypeScript/Electron layer over IPC. The naive approach — JSON-serialized messages over standard IPC channels — becomes a bottleneck when indexing events are high-frequency (file changes in a monorepo) or when returning large search results (top-N code chunks). Electron's IPC uses the Structured Clone Algorithm, which serializes on both sides. With large payloads, this causes garbage collection churn, renderer frame drops, and UI jank. Additional pitfalls: `sendSync()` blocks the entire renderer process; newline-framing bugs in BufReader make the channel appear to drop messages; CPU spin-loops when the channel tears down during app quit.

**Warning signs:**
- Typing latency increases when the indexer is running in the background
- Renderer frame rate drops during large search result returns
- Memory usage spikes on every file-save event

**Prevention:**
- Use napi-rs to build the Rust indexer as a native Node.js addon loaded directly into the main process — this eliminates process-boundary serialization for the hot path and allows shared memory access
- For the remaining IPC surface (background worker messages), pass `ArrayBuffer` (transferable) rather than JSON for large byte payloads
- Never use `ipcRenderer.sendSync()` — always use `ipcRenderer.invoke()` (async/await)
- Keep Rust transport-thin: Rust handles I/O and lifecycle; TypeScript owns the RPC protocol and business logic
- Batch file-change events: debounce file-watcher notifications in Rust before emitting to the TypeScript layer (e.g., 50ms debounce window)

**Phase:** Phase 3 (Indexing/IPC bridge). Benchmark IPC throughput before any feature uses it at scale.

---

### 7. Context Window Degradation at 60% Utilization

**What goes wrong:** AI response quality degrades noticeably at ~60% context utilization — well before the hard limit. The model doesn't announce this degradation. Instead it silently starts forgetting earlier instructions, repeating rejected suggestions, missing established coding patterns, and producing less coherent multi-file edits. For an IDE that accumulates context across a session (open files, conversation history, indexed results, CLAUDE.md), this threshold can be hit within 30–45 minutes of a typical coding session. The "kitchen sink" failure mode: the IDE appends every open file and recent AI turn to every request, ballooning context without adding signal.

**Warning signs:**
- Users report that Claude "forgot" a style rule it was following earlier in the same session
- Multi-file refactors start producing inconsistent output in later files
- Conversation history alone exceeds 50K tokens after an hour of use

**Prevention:**
- Implement smart context selection from the start: score candidate context items (files, history turns, indexed results) by relevance to the current query; include only the top-K by score
- Treat conversation history as a sliding window, not an append-only log; compress older turns into summaries using a cheap summarization call
- Provide a visible "context meter" in the UI — users should see how full the context window is and be able to clear it with one click
- Reserve a minimum headroom budget (e.g., 20% of context window) for the model's output tokens; never allow input to fill >80% of the window

**Phase:** Phase 2 (AI Panel). Context management must be designed into the panel architecture, not bolted on later.

---

### 8. macOS Notarization and Cross-Platform Code Signing Pipeline

**What goes wrong:** macOS 10.15+ requires both code signing AND notarization for apps distributed outside the Mac App Store. Getting this wrong means users see the "Apple cannot check it for malicious software" dialog and the app is blocked from running. The two certificate types are frequently confused: Developer ID Certificate (for direct distribution) vs. Mac App Distribution Certificate (for App Store only). Windows signing is separately complex: Azure Trusted Signing (as of October 2025) requires 3+ years of verifiable business history and is US/Canada-only, meaning new organizations may need an EV code signing certificate from a third-party CA — with some vendors requiring notarized paperwork by mail. Building cross-platform signing into CI late means it becomes a launch blocker.

**Warning signs:**
- Local dev builds work but the CI-built binary is rejected by macOS Gatekeeper
- The CI environment stores signing credentials in plaintext config files
- Windows users report SmartScreen "Windows protected your PC" warnings (unsigned binary)

**Prevention:**
- Wire up signing and notarization in CI (GitHub Actions) in Phase 1 — treat it as infrastructure, not a pre-launch task
- Use separate certificate types per platform: Developer ID Application + Developer ID Installer for macOS direct distribution; EV Code Signing certificate for Windows
- Store all signing credentials as encrypted CI secrets, never in repository files
- Use `electron-builder` with `afterSign` hook for notarization via `@electron/notarize`; ensure `hardened-runtime` entitlements are set (required for notarization since 2019)
- Test the full notarization pipeline on an early Alpha build — Apple's notarization servers occasionally reject builds for entitlement misconfigurations that only show up in production signing

**Phase:** Phase 1 (Distribution setup) for pipeline; Phase 4 (Distribution) for full validation.

---

### 9. Persistent Memory Context Pollution

**What goes wrong:** Persistent memory (conversation history, project notes, learned context) is valuable when relevant and toxic when stale. Teams building AI memory systems consistently hit two failure modes: (1) the entire history is stuffed into the context window on every request, which hits token limits and degrades quality; (2) old, wrong, or contradicted memories surface alongside current context and confuse the model. The second failure is harder to detect because the model doesn't flag contradictions — it attempts to reconcile them, producing plausible but subtly wrong output. A project note from 3 months ago saying "use Redux" still loads when the project has since migrated to Zustand.

**Warning signs:**
- Users notice Claude citing patterns or decisions from previous projects in the current one
- Token usage spikes on project open (history being loaded)
- Memory grows unboundedly — no pruning mechanism exists

**Prevention:**
- Tag every memory item with: timestamp, source (conversation/user-explicit/inferred), project scope, confidence level
- Implement relevance-scored retrieval: load memories based on semantic similarity to current query, not all memories unconditionally
- Support explicit memory management: users can pin, delete, or override memory items
- Implement TTL or "freshness decay" for inferred memories — memories not confirmed or referenced in N days are downranked
- Separate "facts" (user-confirmed, high confidence) from "inferences" (model-derived, lower confidence) in the memory schema

**Phase:** Phase 4 (Persistent Memory). Schema design must account for these failure modes from day one.

---

### 10. Electron Bundle Size and Startup Time

**What goes wrong:** Electron ships a full copy of Chromium and Node.js with every app. A naive Electron build for a VS Code fork easily reaches 300–500 MB installed. VS Code itself is ~350 MB. Adding the Rust indexer binary, native addons, and project assets compounds this. On startup, the renderer process parses and executes thousands of JavaScript modules. Typical Electron apps take 1–2 seconds to open on mid-range laptops, compared to <500ms for Tauri. For users used to snappy IDEs, startup drag creates a poor first impression and is difficult to optimize after the architecture is set.

**Warning signs:**
- `npm run build` produces a bundle >200 MB before assets
- Cold startup time in development exceeds 3 seconds
- App relaunch after update feels slow even though no code changed

**Prevention:**
- Profile and baseline startup time in Phase 1 before adding features — set a target (e.g., <1.5s to interactive on a MacBook M1)
- Use Electron's `app.asar` packaging and enable `asar` compression
- Lazy-load AI panel and indexer modules — the base editor should be functional before AI components initialize
- The Rust indexer binary should start asynchronously on background thread, never blocking the main window render
- Evaluate `electron-vite` for faster build times and better tree-shaking vs. the default webpack bundler VSCodium uses

**Phase:** Phase 1 (Fork Setup). Baseline must be established early; startup regressions are hard to reverse.

---

## Minor Pitfalls
*(Annoying but recoverable)*

---

### 11. Open VSX Extension Gap

**What goes wrong:** Open VSX has substantially fewer extensions than the Microsoft Marketplace. Some popular extensions simply don't exist on Open VSX, or lag behind by several versions. Users migrating from VS Code will hit missing extensions immediately.

**Prevention:** Maintain a documented compatibility matrix. Work proactively with extension authors to publish to Open VSX. For critical gaps (Python, C/C++), identify open-source alternatives (Pyright, clangd) and pre-install them.

**Phase:** Phase 1 (Fork Setup). Document limitations before marketing.

---

### 12. File Watcher Limits on Linux

**What goes wrong:** Linux's `inotify` defaults to 8,192 watched directories. Large monorepos exceed this. The Rust file watcher (`notify` crate) will silently fail to watch new directories unless `fs.inotify.max_user_watches` is increased. Users see stale index for new files in large repos.

**Prevention:** Detect `inotify` limit exhaustion at startup and surface a clear error with fix instructions. Include setup documentation for the sysctl change. Consider using `fanotify` (kernel 5.1+) for recursive watching without per-directory limits.

**Phase:** Phase 3 (Indexing). Add detection before shipping Linux builds.

---

### 13. Windows Long Path Issues

**What goes wrong:** Windows has a default 260-character path limit (MAX_PATH). Deep monorepos (e.g., `node_modules` inside `node_modules`) and long project paths hit this limit, causing Rust file I/O errors that surface as silent indexing failures.

**Prevention:** Enable long path support in the Windows installer (registry key `LongPathsEnabled`). Use `\\?\` extended path prefix in Rust file I/O code on Windows. Test indexing against a deliberately deep directory structure in CI.

**Phase:** Phase 3 (Indexing).

---

### 14. API Key Storage Security on Linux

**What goes wrong:** Electron's `safeStorage` on Linux requires a running D-Bus secret service (KDE Wallet or GNOME Keyring). On headless systems, CI environments, and minimal Linux installs, no such service exists. `safeStorage.isEncryptionAvailable()` returns false, and a naive implementation falls back to plaintext storage.

**Prevention:** Check `safeStorage.isEncryptionAvailable()` explicitly on startup. On Linux without a secret service, warn the user and offer a passphrase-encrypted fallback file store rather than silently writing plaintext. Never fall back to plaintext without user acknowledgment.

**Phase:** Phase 2 (Auth).

---

### 15. Squirrel.Windows Delta Update File-Lock Bug

**What goes wrong:** On first launch after a Squirrel.Windows install, Squirrel holds a file lock on the app binary. If the app checks for updates immediately on launch (a common pattern), the update check fails silently. Additionally, delta package generation requires the previous build artifacts to still exist in the output directory — a CI `clean` step before build destroys them, causing every update to ship a full package rather than a delta.

**Prevention:** Delay update checks on first launch (check for `--squirrel-firstrun` argv flag or use a 10-second startup delay). Configure CI to retain previous build artifacts for delta generation.

**Phase:** Phase 5 (Distribution).

---

## Lessons From Existing AI IDEs

### Cursor

**What they got right:**
- Fast, polished UI that didn't feel like a bolt-on; the fork gave them control over the full surface area
- Inline diff review with accept/reject is the correct UX model — users want to stay in the editor flow, not switch to a chat pane

**What went wrong:**
- Reverse-proxied Microsoft Marketplace requests in violation of ToS; Microsoft enforced it with a runtime check in April 2025, breaking C/C++ extension for Cursor users and creating a PR crisis
- Credit-based billing transition in 2025 caused unexpected bills in the hundreds of dollars — users felt blindsided by token costs they couldn't predict
- Index re-indexing every 10 minutes created a window where phantom APIs surfaced confidently
- Automatically invoking Claude's 1M-token context mode (August 2025) caused unexpected AWS Bedrock rate limit hits and 10× cost spikes for affected users

**Lesson for this project:** Never touch Microsoft's marketplace. Design billing/usage transparency from day one. Never auto-escalate to a larger/more expensive context mode without user consent.

---

### Windsurf (Codeium)

**What they got right:**
- Cascade (agentic multi-step) was ahead of Cursor on autonomous task completion
- Good context-awareness within a session

**What went wrong:**
- Exposed `.env` secret keys in AI completions — a serious security incident where the model would autocomplete secrets from environment files
- Made runtime assumptions (assumed a web server was running, defaulted to localhost preview) that confused users without that context
- Acquired by Cognition in July 2025 after Google poached the CEO/co-founder for $2.4B — organizational instability became a user concern

**Lesson for this project:** Never include `.env`, `.secret*`, or credential files in any context sent to the model. Scan context before sending. Make no runtime assumptions about user environment.

---

### GitHub Copilot (VS Code Extension Path)

**What they got right:**
- Zero friction install (existing VS Code, no migration)
- No marketplace restrictions (it's a first-party Microsoft product)

**What went wrong:**
- 90-second+ spin-up times for the web-based Copilot agent in January 2026 reports
- Acceptance rate of 35–40% (vs. Cursor's 42–45%) — quality issues attributed to multi-model approach and lack of project context
- Tasks touching 10+ files with architectural scope produced noticeably more errors than competitors

**Lesson for this project:** Deep project context (the thing this project is explicitly building) is the real differentiator. Generic completions without codebase understanding are table stakes that every competitor already has.

---

### Augment Code (Extension, Not Fork)

**What they got right:**
- Chose not to fork, using VS Code extension API instead — no marketplace restrictions, no rebase tax, users keep existing VS Code settings and extensions
- Real-time codebase index with semantic understanding of commit history and team conventions

**What went wrong:**
- Extension API limits mean they can't control the full UX — can't rebrand, can't own the AI panel surface area, can't intercept editor internals
- Some deep context features require VS Code private APIs that change without notice

**Lesson for this project:** The fork decision is correct for full UX control, but the rebase tax and marketplace restrictions are real costs that must be budgeted into every sprint.

---

## Phase-Specific Warning Summary

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Fork setup | Marketplace lockout + rebase tax | Open VSX from day 1; minimal diff surface |
| Fork setup | Startup time regression | Baseline <1.5s in Phase 1 CI |
| Auth | PKCE interception via custom URI scheme | Use 127.0.0.1 loopback, not custom protocol |
| Auth | Token storage fails on Linux headless | Check `safeStorage.isEncryptionAvailable()` |
| AI Panel | Context window degradation at 60% | Smart context selection + compaction from day 1 |
| AI Panel | Token burn rate / quota exhaustion | Prompt caching + tiered context + token meter |
| Indexing | Stale index serving phantom APIs | File-watcher-driven incremental updates |
| Indexing | Memory blowup on large repos | Two-tier index; background full-embed |
| Indexing | IPC serialization bottleneck | napi-rs native addon for hot path |
| Indexing | Linux inotify limits | Detect and surface at startup |
| Persistent Memory | Context pollution from stale memories | Relevance scoring + TTL decay |
| Distribution | macOS notarization misconfiguration | Wire signing pipeline in Phase 1 CI |
| Distribution | Windows Squirrel file-lock on first launch | 10-second update-check delay |
| Distribution | Secret key exposure in AI context | Block env/secret files from context assembly |

---

## Sources

- [Augment Code — To Fork or Not to Fork](https://www.augmentcode.com/blog/to-fork-or-not-to-fork)
- [The Register — Microsoft subtracts C/C++ extension from VS Code forks (April 2025)](https://www.theregister.com/2025/04/24/microsoft_vs_code_subtracts_cc_extension/)
- [DevClass — VS Code extension marketplace wars: Cursor users hit roadblocks (April 2025)](https://www.devclass.com/development/2025/04/08/vs-code-extension-marketplace-wars-cursor-users-hit-roadblocks/1629343)
- [The Register — Anthropic admits Claude Code quotas running out too fast (March 2026)](https://www.theregister.com/2026/03/31/anthropic_claude_code_limits/)
- [Cursor forum — Claude 4.5 long-context mode cost spike](https://forum.cursor.com/t/cursor-automatically-invokes-claude-4-5-long-context-mode-context-1m-2025-08-07-increasing-costs-and-hitting-bedrock-rate-limits/139092)
- [Morphik — Codebase Indexing in AI Tools](https://www.morphllm.com/codebase-indexing)
- [Augment Code — Real-time index for your codebase](https://www.augmentcode.com/blog/a-real-time-index-for-your-codebase-secure-personal-scalable)
- [Kilo Code — Codebase Indexing running from scratch issue #4080](https://github.com/Kilo-Org/kilocode/issues/4080)
- [Context window management — 50 sessions](https://blakecrosley.com/blog/context-window-management)
- [MindStudio — Claude Code context window limit management](https://www.mindstudio.ai/blog/claude-code-context-window-limit-management)
- [Electron — Deep Links](https://www.electronjs.org/docs/latest/tutorial/launch-app-from-url-in-another-app)
- [Electron — Code Signing](https://www.electronjs.org/docs/latest/tutorial/code-signing)
- [Security Boulevard — Code signing Electron for macOS (December 2025)](https://securityboulevard.com/2025/12/how-to-code-signing-an-electron-js-app-for-macos/)
- [NAPI-RS GitHub](https://github.com/napi-rs/napi-rs)
- [Eclipse Foundation — Why Cursor/Windsurf fork VS Code but shouldn't](https://blogs.eclipse.org/post/thomas-froment/why-cursor-windsurf-and-co-fork-vs-code-shouldnt)
- [Visual Studio Magazine — VS Code Fork Wars (January 2026)](https://visualstudiomagazine.com/articles/2026/01/26/what-a-difference-a-vs-code-fork-makes-antigravity-cursor-and-windsurf-compared.aspx)
- [Ministry of Testing — Lessons in quality engineering from Cursor and Windsurf](https://www.ministryoftesting.com/articles/lessons-in-quality-engineering-from-working-with-cursor-and-windsurf)
- [ghuntley — Visual Studio Code is designed to fracture](https://ghuntley.com/fracture/)
- [Electron autoUpdater — Squirrel.Windows](https://www.electron.build/squirrel-windows.html)
