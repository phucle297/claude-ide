# Claude Code IDE

## What This Is

A VS Code fork (desktop app) built exclusively for Claude Code power users — solo developers and engineering teams who want a first-class IDE experience around Claude's AI capabilities. Users authenticate via Anthropic OAuth or an API key and get a purpose-built workspace with deep, persistent project context that survives across sessions.

## Core Value

A developer opens a project and Claude already knows the full codebase, remembers every prior conversation, and picks the right context automatically — no explaining, no copy-pasting, no setup.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] VS Code fork with custom branding and Claude-optimized UI/UX
- [ ] Anthropic OAuth login (users connect their existing Anthropic subscription)
- [ ] API key auth (fallback — users enter their own API key)
- [ ] Persistent memory across sessions (Claude remembers context from prior work)
- [ ] Automatic codebase indexing (IDE scans and understands the full repo on open)
- [ ] AI conversation history per project (searchable, persistent across sessions)
- [ ] Smart context inclusion (IDE auto-selects relevant files when a question is asked)
- [ ] macOS distribution
- [ ] Windows distribution
- [ ] Linux distribution
- [ ] Integrated Claude Code panel (chat, inline edits, diff review, accept/reject)

### Out of Scope

- Support for other AI providers (GPT, Gemini, etc.) — this is a Claude-only IDE by design
- Custom AI model training or fine-tuning — out of scope, use Anthropic's APIs
- Cloud IDE / browser-based version — desktop-first for v1, web is a future milestone
- Proprietary subscription/billing — users pay Anthropic directly, not us

## Context

- **Competitive landscape**: Cursor and Windsurf have fragmented multi-model support. Neither is optimized for Claude Code's specific workflow (the `claude` CLI, slash commands, hooks, MCP servers, etc.). This is the gap.
- **Target users**: Developers who are already using Claude Code via terminal and find the CLI-only UX limiting. Also engineering teams who want shared AI project context.
- **Claude Code CLI**: The Anthropic `claude` CLI is the execution engine. The IDE decision on whether to wrap the CLI or reimplement is deferred to the phase that builds the AI panel — pick whatever gives the best UX.
- **Auth model**: Zero cost to the product builder. Users supply their Anthropic credentials (subscription or API key). The IDE acts as an authenticated client, not a proxy.

## Constraints

- **Tech stack**: Rust + TypeScript — Rust for performance-sensitive work (indexing, file watching), TypeScript for UI and IDE layer (consistent with VS Code's own codebase)
- **Platform**: Ship simultaneously on macOS, Windows, Linux — cross-platform from day one
- **Fork base**: VSCodium (open-source VS Code, no Microsoft telemetry) or Monaco — decision deferred to Phase 1 based on fastest path to MVP
- **Auth**: Must support both Anthropic OAuth and raw API key — no user left behind regardless of how they access Claude

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| VS Code fork over extension | Extensions can't control the full UX, can't rebrand, can't own the surface area needed for deep context features | — Pending |
| Zero-cost-to-builder auth model | Users already pay Anthropic; adding another subscription layer creates friction and churn risk | — Pending |
| Claude-only (no multi-model) | Focus beats flexibility for a v1. Multi-model support fragments context design and dilutes the brand promise | — Pending |
| Rust + TypeScript stack | Rust handles indexing performance (potentially millions of tokens of codebase), TS keeps IDE layer compatible with VS Code ecosystem | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-19 after initialization*
