---
name: anthropic-sdk
description: Patterns for the Anthropic SDK integration in Claude Code IDE — streaming, prompt caching, context assembly
type: skill
---

# Anthropic SDK Patterns

Use when writing or modifying the Claude API client in the Extension Host.

## Model selection

Default to `claude-sonnet-4-6` for interactive chat. Use `claude-opus-4-7` only for tasks where quality clearly justifies higher cost (e.g. complex multi-file refactors). Never use a retired model.

## Streaming (always use)

All chat responses must stream — buffered responses feel slow in an IDE context:

```typescript
const stream = await client.messages.stream({
  model: 'claude-sonnet-4-6',
  max_tokens: 8096,
  system: [{ type: 'text', text: systemPrompt, cache_control: { type: 'ephemeral' } }],
  messages,
});

for await (const chunk of stream) {
  if (chunk.type === 'content_block_delta' && chunk.delta.type === 'text_delta') {
    panel.appendText(chunk.delta.text);
  }
}
```

## Prompt caching (required for large contexts)

Always cache the system prompt and any static context prefix. Cache hits reduce latency and cost by ~90%:

```typescript
// Static prefix (system prompt + codebase summary) — cache this
{ type: 'text', text: systemPrompt, cache_control: { type: 'ephemeral' } }

// Dynamic suffix (user message + retrieved chunks) — do NOT cache, changes every turn
{ type: 'text', text: userMessage }
```

The `ephemeral` cache has a 5-minute TTL — sufficient for interactive sessions.

## Context assembly order

Assemble context in this order to maximize cache hit rate (stable content first):

1. System prompt (cached)
2. CLAUDE.md content (cached)
3. Conversation history (partially cached — only past turns)
4. Retrieved code chunks from indexer (changes per query — not cached)
5. User message (not cached)

## Token counting

Always show the user the token count + USD estimate before and after each request. Use `client.messages.countTokens()` for pre-request estimates:

```typescript
const { input_tokens } = await client.messages.countTokens({ model, system, messages });
const estimatedCost = (input_tokens / 1_000_000) * 3.0; // Sonnet input rate
```

## Error handling

Surface API errors to the user with actionable messages — never swallow them silently:

- `AuthenticationError` → prompt re-authentication
- `RateLimitError` → show retry countdown
- `OverloadedError` → retry with exponential backoff (max 3 attempts)
