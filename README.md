# Watchy Agent Skills

Standalone distribution bundle for the public Watchy skills repository.

Intended GitHub remote:

```bash
https://github.com/watchy-dev/watchy
```

## Install

```bash
npx skills add watchy-dev/watchy
```

Install a single skill:

```bash
npx skills add watchy-dev/watchy --skill watchy-error-capture
npx skills add watchy-dev/watchy --skill watchy-mcp-setup
```

## Contents

- `AGENTS.md`: skill discovery and install metadata
- `.claude-plugin/`: Claude-compatible marketplace metadata
- `.cursor-plugin/`: Cursor-compatible metadata
- `skills/`: published skill payloads

## Publish

Push the contents of this directory as the root of the public `watchy-dev/watchy` repository.

Before publishing:

1. Confirm the GitHub repository is public.
2. Confirm anonymous clone works for `https://github.com/watchy-dev/watchy.git`.
3. Run `npx skills add . --list` from this directory.
