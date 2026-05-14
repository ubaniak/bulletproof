# bulletproof

A Claude Code skill encoding a domain-driven backend architecture pattern: clear separation between entities, storage adapters, third-party gateways, usecase/service logic, and transport handlers.

## What it does

When this skill is installed, Claude Code will use the bulletproof layout to:

- Scaffold new domains with the correct layer structure
- Review existing code against the layering/import rules
- Flag DTO leaks, layer violations, and god-usecases
- Wire `setup` composition roots correctly

The skill auto-triggers when you mention "bulletproof", "domain layout", "usecase", "gateway", "storage adapter", or ask how to structure a service.

## Install

Clone into your Claude skills directory:

```bash
git clone <this-repo-url> ~/.claude/skills/bulletproof
```

Or symlink an existing checkout:

```bash
ln -s /path/to/this/repo ~/.claude/skills/bulletproof
```

Restart Claude Code to pick up the new skill.

## Usage

In any Claude Code session, ask things like:

- "Scaffold a new `billing` domain using bulletproof"
- "Review this code against the bulletproof layout"
- "Where should this DTO live?"
- "Is it OK for the `orders` domain to import `payments/storage`?"

## Files

- [`SKILL.md`](./SKILL.md) — skill definition loaded by Claude Code
- [`layout.md`](./layout.md) — original layout sketch this skill is built from

## Pattern summary

```
<domain>/
    entities/
    storage                       # interface
    storage/<impl>                # adapter
    storage/<impl>_dto            # translation
    gateway                       # interface
    gateway/<impl>                # adapter
    gateway/<impl>_dto            # translation
    usecase                       # business logic
    transport                     # interface
    transport/<impl>              # handler
    transport/<impl>_dto          # translation
    setup                         # wiring
```

Every adapter `<impl>` pairs with a sibling `<impl>_dto`.

**Hard rule**: domains may only import another domain's `entities` and `usecase`.

See [`SKILL.md`](./SKILL.md) for full rules, scaffold checklist, review checklist, and anti-patterns.
