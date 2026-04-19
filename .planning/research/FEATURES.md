# Features Research — Claude Code IDE

**Domain:** AI-native desktop IDE (VS Code fork, Claude-only)
**Researched:** 2026-04-19
**Sources:** Cursor docs, Windsurf review, Claude Code CLI docs (code.claude.com), RedMonk developer survey, community analysis

---

## Table Stakes
(Users expect these — ship without them and users leave)

### Core Editor Experience
- Syntax highlighting + language server support (LSP) — complexity: low — inherited from VS Code fork; non-negotiable baseline
- Multi-tab file editing — complexity: low — VSCodium provides this; do not remove or regress
- Integrated terminal — complexity: low — critical: Claude Code CLI runs here; must be low-latency (Cursor's terminal input lag is a documented complaint)
- Git integration (status, diff, staging) — complexity: low — VSCodium provides; keep it working with Claude-aware diff views
- File explorer with search — complexity: low — standard; VSCodium provides
- Extensions / plugin system — complexity: medium — users expect VS Code extensions to work; VSCodium compatibility is the target

### AI Chat Panel
- Inline chat / ask about selected code — complexity: low — Cursor's Cmd+K equivalent; users expect this by default
- Multi-file edit (agentic) — complexity: medium — Cursor Composer and Windsurf Cascade both do this; users now expect cross-file edits from a single prompt
- Accept / reject diff UI — complexity: medium — show proposed changes as diffs; user confirms before applying; critical for trust
- Conversation history per session — complexity: low — within-session scrollback; must work before cross-session persistence
- Code references in chat (@ file, @ symbol) — complexity: low — Cursor pioneered this; users expect to @ a file and have it included in context

### Authentication
- Anthropic OAuth login — complexity: medium — users authenticate with existing Anthropic account; no separate billing layer
- API key fallback — complexity: low — paste key, start working; no OAuth required
- Session persistence (stay logged in) — complexity: low — token refresh, keychain storage

### Autocomplete
- Inline completions (ghost text) — complexity: medium — single-line and multi-line; Cursor Tab is the market leader here; absence is conspicuous
- Completion accepts with Tab — complexity: low — standard UX contract users won't break from

### Stability and Performance
- Indexing that doesn't freeze the UI — complexity: high — Cursor users complain about RAM usage on large repos; background indexing with progress indicator is required
- Predictable token/cost visibility — complexity: medium — 2025 market research: opaque pricing is the #1 complaint across Cursor and Windsurf; show estimated cost per request

---

## Differentiators
(Competitive advantages — this is where the product wins)

### Persistent Context (Core Differentiator)
- Cross-session conversation history per project — complexity: high — why this matters: Claude Code CLI sessions are ephemeral; the IDE stores every conversation, searchable, linked to the project; Cursor's memory is per-machine and manual
- Auto memory surfacing — complexity: high — Claude Code has an auto-memory system (`~/.claude/projects/<repo>/memory/MEMORY.md`) that accumulates learnings; the IDE should visualize this and let users view/edit without leaving the editor; no competitor exposes this
- CLAUDE.md editor and scaffolding — complexity: medium — CLAUDE.md is the primary persistent instruction file; the IDE should surface it prominently (not buried in the file tree), offer a guided setup via `/init`, and warn when it exceeds the 200-line context limit; Cursor and Windsurf have no equivalent concept
- Context window visualization — complexity: medium — Claude Code's official docs include a context window visualization; surfacing "how full is the context right now" and "what's loaded" builds trust and lets power users optimize; no competitor does this clearly
- `.claude/rules/` management UI — complexity: medium — path-scoped rules (loaded only when matching files open) are a power feature; a visual rule editor with file-glob targeting and live preview of which rules are active is a real DX improvement over editing JSON and markdown by hand

### Claude Code CLI Workflow Integration
- CLAUDE.md multi-level hierarchy visualization — complexity: medium — Claude Code supports managed-policy / project / user / local CLAUDE.md layers; the IDE should show which layer is active and allow editing each layer without file navigation
- Hooks editor and live log viewer — complexity: high — hooks (PreToolUse, PostToolUse, SessionStart, etc.) are JSON-configured shell/HTTP/LLM handlers; the IDE should provide a visual hook editor (no raw JSON required) and a live log showing which hooks fired, their exit codes, and any blocked operations; power feature that Cursor/Windsurf have zero equivalent of
- MCP server management — complexity: medium — Model Context Protocol servers connect Claude to GitHub, databases, Slack, etc.; the IDE should list configured MCP servers, show their status, and let users add/remove them through a UI rather than editing settings.json; Cursor has plugins but no MCP-native management
- Slash commands / skills browser — complexity: low — `.claude/skills/` contains reusable prompt workflows; the IDE should surface these as a discoverable palette (like VS Code's command palette) with descriptions and invocation shortcuts
- Session management (resume, fork, rename) — complexity: medium — Claude Code CLI supports named sessions, resumption by name or ID, and forking; the IDE should expose this as a session browser in the sidebar, not just a terminal flag
- Permission mode selector — complexity: low — Claude Code has `default`, `acceptEdits`, `plan`, `auto`, `bypassPermissions` modes; a persistent status-bar indicator and one-click switcher is clearer than passing flags; surfaces a unique Claude Code concept visually

### Codebase Intelligence
- Automatic codebase indexing on project open — complexity: high — background AST/semantic index of the full repo; enables smart context selection without manual @ references; Cursor does this but users complain about RAM; implement with progress visibility and cancellation
- Smart context selection — complexity: high — when a user asks a question, auto-select the most relevant files/symbols rather than requiring manual @ references; the core "Claude already knows your codebase" promise; requires the index
- Semantic search across the codebase — complexity: high — natural language search ("find where auth tokens are validated") instead of regex grep; differentiated UX for code navigation

### Team Collaboration
- Shared CLAUDE.md via version control — complexity: low — the IDE should make it obvious that the project-level CLAUDE.md is committed to git and shared with the team; first-time setup should prompt to commit it
- Team skills library — complexity: medium — `.claude/skills/` is already shareable via git; the IDE should show which skills came from the project repo vs personal vs installed plugins, and allow team leads to pin recommended skills
- Org-wide managed CLAUDE.md support — complexity: medium — Claude Code supports a managed-policy CLAUDE.md at the OS-level path; for enterprise teams using MDM/Ansible, the IDE should display this layer as read-only with a clear "managed by your organization" label
- Agent Teams dashboard (experimental) — complexity: high — Claude Code's experimental multi-agent team feature (lead + teammates with shared task list) benefits from a visual dashboard showing each agent's state, current task, and messages; CLI tmux split-panes is the current UX; an integrated panel is strictly better; note: this feature is experimental as of v2.1.32
- Conversation history search across projects — complexity: medium — searchable history across all project sessions; "find the conversation where we discussed the auth redesign" is a real workflow; no competitor offers cross-project AI conversation search

### Developer Experience Gaps (Cursor/Windsurf complaints addressed)
- Transparent per-request cost display — complexity: medium — token count + estimated USD cost shown per message; addresses the #1 billing complaint across all AI IDEs (Cursor's "$200 in 5 days" stories)
- Configurable spending limits — complexity: medium — per-session and per-day hard caps with warnings before hitting them; predictability beats surprise invoices
- Inline permission approval (no context switch to terminal) — complexity: medium — Claude Code's "can I edit this file?" prompts currently happen in the terminal; in the IDE, these should appear as non-blocking inline banners or diff previews, not blocking modal dialogs
- Low-latency terminal — complexity: medium — Claude Code's primary interface is the terminal; the integrated terminal must have < 50ms input latency; Cursor's terminal lag is a documented user complaint to avoid repeating

---

## Anti-Features
(Things that look good but hurt — deliberately avoid)

- **Multi-model support** — users must choose models, context strategies differ, brand promise dilutes; the CLI-first users already chose Claude; adding GPT/Gemini adds maintenance burden and creates support surface for model-specific bugs with no market differentiation
- **Aggressive ghost-text autocomplete on every keystroke** — documented as disruptive; Cursor users turn this off; interrupts flow; default should be opt-in or triggered only on pause; token cost is also 15x higher than necessary (3,000 tokens per completion vs ~200 needed)
- **Opaque pricing / surprise billing** — the market has learned this causes immediate cancellations; never hide token consumption; show it always
- **Forced onboarding flows that block coding** — users want to be in code immediately; multi-step onboarding wizards before first session create churn; use progressive disclosure (suggest CLAUDE.md setup after first session, not before)
- **AI suggestions that modify files without diff preview** — any agentic file edit must show a diff and require accept/reject; silent file modification destroys trust and is the leading cause of "I stopped using it" reports
- **Mandatory cloud sync of AI context/memory** — auto memory is currently machine-local by design; users explicitly want privacy-respecting, local-first memory; forcing cloud sync will cause enterprise/security-conscious users to leave
- **Bundled Anthropic subscription billing** — the project design correctly excludes this; users pay Anthropic directly; adding a proprietary billing layer creates a duplicate cost and a reason for refunds; do not build this
- **Per-message loading spinners that block the editor** — AI responses are streamed; the editor must remain fully usable while a response is generating; blocking the UI during inference is a table-stakes failure
- **Generic AI "explain code" / "write tests" buttons on the toolbar** — surface-level feature appearance that power users immediately ignore; wastes toolbar space; Claude Code users prefer conversational interaction over one-click magic buttons

---

## Claude Code-Specific Features
(Unique to Claude Code CLI workflows — not present in Cursor/Windsurf)

- **CLAUDE.md hierarchy editor** — what it unlocks: Claude Code has four CLAUDE.md scopes (managed-policy, project, user, local); visualizing the active hierarchy and editing each layer in-context is a unique surface area no general AI IDE provides; teams can see exactly what instructions are in effect
- **Auto memory viewer and editor** — what it unlocks: Claude's `MEMORY.md` accumulates learnings from corrections automatically (`~/.claude/projects/<repo>/memory/`); surfacing this as an editable sidebar panel lets users audit, correct, and curate what Claude has learned; no competitor has this concept
- **Hooks visual editor with lifecycle log** — what it unlocks: hooks fire at PreToolUse, PostToolUse, SessionStart, SessionEnd, Stop, UserPromptSubmit, TeammateIdle, TaskCreated, TaskCompleted (12 total lifecycle events); a visual editor replaces raw JSON editing; a live log shows what fired and why; this is the safety and automation control panel for power users
- **MCP server registry and status panel** — what it unlocks: MCP is now the standard integration layer (GitHub, Slack, databases); a visual panel to install, configure, and monitor MCP servers (with connection status) is cleaner than CLI `claude mcp` commands; supports the ecosystem of 100s of available MCP servers
- **Slash commands / skills palette** — what it unlocks: `.claude/skills/` contains team-defined reusable workflows; surfacing them in a searchable palette (like VS Code's Cmd+P) with descriptions, usage examples, and the ability to create new skills inline means teams can encode and share institutional knowledge
- **Session browser with fork and resume** — what it unlocks: `claude --resume <name>` and `claude --fork-session` are powerful for managing parallel work streams; a visual session browser with named sessions, creation timestamps, and fork lineage replaces flag memorization
- **Permission mode status bar indicator** — what it unlocks: `default / acceptEdits / plan / auto / bypassPermissions` are five distinct operational modes; a persistent status-bar chip showing current mode + one-click switching makes a confusing CLI concept immediately understandable and controllable
- **Agent Teams dashboard** — what it unlocks: experimental multi-agent coordination with lead + teammates, shared task list, and inter-agent messaging; current UX requires tmux panes; an integrated IDE panel showing each agent's state, task, and messages makes this feature accessible beyond power users; requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
- **Worktree-per-session launcher** — what it unlocks: `claude --worktree <name>` creates isolated git worktrees for parallel work without branch conflicts; the IDE should offer a "new session in isolated worktree" flow that creates the worktree, opens it in a tab, and links the session — replacing the multi-terminal juggling this currently requires
- **`/init` guided project setup wizard** — what it unlocks: `/init` analyzes a codebase and generates CLAUDE.md, skills, and hooks automatically; in the IDE, this becomes an onboarding flow for new projects that feels native rather than a CLI command to remember
- **Context window budget indicator** — what it unlocks: Claude Code's context window has a defined budget; CLAUDE.md, auto memory, conversation history, and indexed files all consume it; a visual token budget bar (showing what's loaded and how much space remains) turns an invisible resource into something manageable; Anthropic's own docs include this visualization

---

## Feature Dependencies
(What needs to be built before what)

```
[Auth: OAuth + API key]
    └─► [IDE Shell: VS Code fork, branding, extension compat]
            └─► [Integrated Terminal: low-latency, Claude CLI passthrough]
                    └─► [Basic AI Chat Panel: inline diffs, accept/reject]
                            └─► [Session Persistence: named sessions, resume, fork]
                                    └─► [Cross-session Conversation History: per-project, searchable]

[IDE Shell]
    └─► [Codebase Indexer: background AST/semantic index]
            └─► [Smart Context Selection: auto @ from index]
                    └─► [Semantic Codebase Search: NL queries over index]

[Basic AI Chat Panel]
    └─► [CLAUDE.md Editor: create, edit, hierarchy view]
            └─► [Auto Memory Viewer: MEMORY.md sidebar panel]
                    └─► [Context Window Budget Indicator: tokens used / remaining]

[Claude CLI Passthrough]
    └─► [Permission Mode Switcher: status bar indicator]
    └─► [Slash Commands / Skills Palette: .claude/skills/ discovery]
    └─► [MCP Server Panel: install, configure, status]
    └─► [Hooks Editor + Lifecycle Log: JSON → visual, live event stream]

[Session Persistence]
    └─► [Session Browser: named sessions, fork lineage, worktree links]
            └─► [Worktree-per-session Launcher: isolated git worktrees]

[Hooks Editor]
    └─► [Agent Teams Dashboard: lead/teammate state, task list, messages]
            (requires CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS — defer until stable)

[Auth: OAuth + API key]
    └─► [Cost Display: per-request token count + USD estimate]
            └─► [Spending Limits: per-session and per-day caps]
```

**Critical path for MVP:**
Auth → IDE Shell → Terminal → Basic Chat (with diff UI) → CLAUDE.md editor → Session persistence

**Power-user phase (after MVP):**
Codebase indexer → Smart context → Hooks editor → MCP panel → Skills palette → Session browser → Auto memory viewer

**Team phase:**
Shared CLAUDE.md workflows → Team skills library → Org-managed policy layer → Agent Teams dashboard (once stable)

---

## Complexity Summary

| Feature | Complexity | Phase |
|---------|-----------|-------|
| Auth (OAuth + API key) | Medium | MVP |
| IDE shell / VS Code fork | High | MVP |
| Low-latency terminal | Medium | MVP |
| Basic chat + diff UI | Medium | MVP |
| CLAUDE.md editor | Medium | MVP |
| Permission mode switcher | Low | MVP |
| Slash commands palette | Low | MVP |
| Session persistence | Medium | MVP |
| Codebase indexer | High | Phase 2 |
| Smart context selection | High | Phase 2 |
| Cross-session history | Medium | Phase 2 |
| Cost display + limits | Medium | Phase 2 |
| Auto memory viewer | Medium | Phase 2 |
| MCP server panel | Medium | Phase 2 |
| Hooks editor + live log | High | Phase 2 |
| Context window indicator | Medium | Phase 2 |
| Semantic codebase search | High | Phase 3 |
| Session browser + worktrees | Medium | Phase 3 |
| Team skills library | Medium | Phase 3 |
| Org-managed CLAUDE.md | Medium | Phase 3 |
| Agent Teams dashboard | High | Phase 4 (post-stable) |
