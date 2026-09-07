# Agent Skills — public collection

This repository contains reusable, vendor-neutral guidance and selected
shareable Agent Skills. It is intentionally separate from any private local
skill distribution, automation state, credentials, logs, backups, and
client-specific material.

## Contents

- [`BEST_PRACTICES.md`](BEST_PRACTICES.md) — how to design compact,
  portable skills with progressive disclosure.
- [`docs/DAILY_SKILL_SYNC.md`](docs/DAILY_SKILL_SYNC.md) — a generalized
  runner-only synchronization pattern for local tools and cloud workspaces.

## Related project

- [Retain](https://github.com/tolmachevmaxim/retain) — a public,
  local-first conversation archive and migration reference for AI coding
  tools.

`BEST_PRACTICES.md` is documentation, not itself an invokable skill. New
shareable skills should use the standard layout:

```text
skills/<skill-name>/SKILL.md
skills/<skill-name>/references/...
skills/<skill-name>/scripts/...
```

For cloud-compatible repository discovery, a project may expose the same
physical skill through repo-scoped mirrors:

```text
.agents/skills/<skill-name> -> ../../skills/<skill-name>
.claude/skills/<skill-name> -> ../../skills/<skill-name>
.cursor/skills/<skill-name> -> ../../skills/<skill-name>
```

Local user-level installations do not automatically appear in a fresh cloud
clone. Cloud workflows therefore need repo-scoped skills or an authenticated
setup step that installs a private bundle without embedding credentials in
Git.

## Public-safety boundary

Only reviewed, anonymized material belongs here. Do not add personal paths,
account identifiers, private repository names, tokens, cookies, raw logs,
backups, client data, or unreviewed skill caches.

The local authoring directory may contain additional private material; its
presence on disk is not evidence that it is part of this public repository.
