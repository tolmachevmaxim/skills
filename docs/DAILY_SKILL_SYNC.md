# Daily Agent Skill Synchronization

This is a generalized public template for synchronizing personal skills and
allowlisted native plugins across local agent hosts and cloud workspaces. It
contains no account-specific paths, repository names, credentials, or runtime
state.

## Distribution model

Keep two layers separate:

1. Personal skills use the portable `SKILL.md` format and may be materialized
   through symlinks or generated repo-scoped mirrors.
2. Native plugins remain namespaced and are installed through their host's
   plugin mechanism. A plugin cache must never be flattened into personal
   skill roots.

Typical local discovery roots are:

| Host | Typical user-level root |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.agents/skills/` (legacy setups may also use `~/.codex/skills/`) |
| Antigravity / Gemini CLI | installation-dependent; commonly `~/.gemini/config/skills/` or `~/.gemini/skills/` |
| Cursor | `~/.cursor/skills/` |

Always verify the active roots for the installed versions. Paths are examples,
not a claim about every installation.

## Required runner workflow

The mutation logic belongs in the maintained runners. A scheduled job should
run the equivalent of:

```bash
python3 scripts/sync_agent_plugins.py --apply --json
python3 scripts/sync_personal_skills.py --apply --json
```

Do not recreate either algorithm with one-off shell scripts. Treat both JSON
outputs, the personal-skill state, backup manifest, JSONL audit log, cloud
report, and project-target report as the source of truth.

If either runner fails, report the exact failure and stop. Do not compensate
with manual deletion, copying, cleanup, GitHub pushes, or target-repository
repair outside the runner flow.

## Verification contract

- Verify every reported personal-skill symlink action by resolving its
  destination and reading `SKILL.md`.
- If there are no actions, run the full state-based audit across all managed
  roots.
- Verify native-plugin parity from the checked-in allowlist. Report native
  plugin versions separately; vendor marketplace versions do not need to
  match.
- Treat an enabled but unclassified plugin as drift. Never auto-import it.
- Keep plugin skills namespaced and preserve hooks, manifests, and lifecycle
  metadata.

## Cloud distribution

Local symlinks do not cross the machine boundary. To make skills available to
cloud coding environments without the local computer, publish a sanitized
physical snapshot to a repository that is appropriate for its sensitivity,
then expose repo-scoped paths such as:

```text
skills/<name>/SKILL.md                 # physical snapshot
.agents/skills/<name> -> ../../skills/<name>
.claude/skills/<name> -> ../../skills/<name>
.cursor/skills/<name> -> ../../skills/<name>
```

The snapshot repository's visibility must be checked explicitly. A private
snapshot needs authenticated cloud setup; a public repository must contain
only reviewed shareable material. Native local plugin installation is not
proof that the plugin is available in a cloud environment. Report cloud
plugin coverage as unverified unless a fresh cloud canary proves it.

## Project targets

For each configured target repository, verify the generated change through the
target's normal review and merge workflow. After publication, read back the
target's default branch and confirm the expected count and paths under every
configured repo-scoped skill root.

## Reporting

Report, separately:

- personal-skill action buckets, conflicts, warnings, manifest, and
  local/cloud/project convergence;
- plugin actions, versions, required-skill counts, unclassified plugins,
  local plugin convergence, and the cloud-plugin boundary.

Never include secrets in reports. A cloud repository is also the practical
bridge for cloud-first vibe coding: the agent can clone the reviewed skill
snapshot from GitHub even when the author's computer is offline.
