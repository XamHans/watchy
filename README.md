# Watchy Agent Skills

Standalone distribution bundle for the public Watchy skills repository.

Intended GitHub remote:

```bash
https://github.com/XamHans/watchy
```

## Install

```bash
npx skills add XamHans/watchy
```

Install a single skill:

```bash
# Error capture setup (generates Lambda integration code)
npx skills add XamHans/watchy --skill watchy

# Error investigation playbook (debug production errors with AI agent)
npx skills add XamHans/watchy --skill watchy-debug
```

## Contents

- `AGENTS.md`: skill discovery and install metadata
- `.claude-plugin/`: Claude-compatible marketplace metadata
- `.cursor-plugin/`: Cursor-compatible metadata
- `skills/`: published skill payloads

## Publish

Push the contents of this directory as the root of the public `XamHans/watchy` repository.

Before publishing:

1. Confirm the GitHub repository is public.
2. Confirm anonymous clone works for `https://github.com/XamHans/watchy.git`.
3. Run `npx skills add . --list` from this directory.
