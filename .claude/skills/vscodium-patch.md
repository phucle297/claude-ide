---
name: vscodium-patch
description: Creating, applying, and maintaining VSCodium patches for the Claude Code IDE fork
type: skill
---

# VSCodium Patch Workflow

Use when creating or updating custom patches to the VSCodium fork base.

## Patch location

All custom patches go in `patches/user/`. The VSCodium build system auto-applies everything in this directory during `npm run build-vscodium`.

## Creating a patch

1. Make changes in the prepared (unpacked) VS Code source tree
2. Generate the patch from the diff:
   ```bash
   git diff HEAD > patches/user/my-feature.patch
   ```
3. Name patches descriptively: `claude-ai-panel.patch`, `product-branding.patch`, `open-vsx-marketplace.patch`

## Rules

- One concern per patch file — don't combine unrelated changes
- Never patch `src/vs/workbench/` internals unless absolutely required; prefer extension API
- After each VSCodium upstream sync, run `npm run build-vscodium` and fix any patch conflicts before merging
- Keep patch files small — large patches are hard to rebase and indicate overreach into internals

## product.json changes (no patch needed)

Branding, marketplace URL, telemetry, and app identity live in `product.json`. Edit directly — VSCodium treats this as the primary override surface:

```json
{
  "nameShort": "Claude Code IDE",
  "nameLong": "Claude Code IDE",
  "applicationName": "claude-code-ide",
  "extensionsGallery": {
    "serviceUrl": "https://open-vsx.org/vscode/gallery",
    "itemUrl": "https://open-vsx.org/vscode/item"
  }
}
```

## Upstream sync cadence

VSCodium syncs monthly with VS Code. When a new VSCodium release drops:
1. Merge/rebase our `patches/user/` onto the new base
2. Verify `product.json` didn't get overwritten
3. Run full build + smoke test before shipping
