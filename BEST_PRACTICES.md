# Skills Best Practices

Universal guide for building context-efficient Agent Skills across **Claude Code, OpenAI Codex, Cursor, Gemini CLI, Antigravity, and compatible hosts**.

**Last Updated:** 2026-08-21

---

## TL;DR — what changed since the open standard

As of **Dec 18, 2025**, Agent Skills are an **open standard** ([agentskills.io](https://agentskills.io), GitHub `agentskills/agentskills`), adopted by **30+ tools** (Claude Code, Codex, Cursor, Gemini CLI, Copilot, Goose, Kiro, JetBrains Junie, Roo Code…).

Practical consequence: **`SKILL.md` is one portable format**. Write the skill once; each vendor adds an optional *sidecar* for its own extras. The old "three divergent platforms" framing is dead.

| Vendor | Native skills? | Install path | Manual invoke | Vendor sidecar / extras |
|--------|---------------|--------------|---------------|--------------------------|
| **Claude Code** | Yes (reference impl) | `~/.claude/skills/` | `/skill-name` | Richest frontmatter; plugins; `context: fork`; hooks |
| **OpenAI Codex** | Yes | `~/.agents/skills/` | `$skill-name` or `/skills` | `agents/openai.yaml` (UI/policy/MCP) |
| **Cursor** | Yes (v2.4+) | `~/.cursor/skills/` | `/skill-name`, `@skill-name` | `paths` glob; `/migrate-to-skills` |
| **Gemini CLI / Antigravity** | Yes | `~/.gemini/skills/` or `~/.gemini/config/skills/` | none — auto via `activate_skill` | bundled inside "extensions" |

> ⚠️ **Correction to the old version of this doc:** the previous "15–25 lines max, >30 = refactor" target was far stricter than reality. The canonical ceiling is **SKILL.md body < 500 lines** (Anthropic, repeated in both best-practices and Claude Code docs). Keep it *as lean as possible* via progressive disclosure — but lean means "push detail to `references/`", not "cram everything into 20 lines."

---

## WARNING: Don't let AI generate skills blindly

**Mistake:** asking an agent to "create a skill" without the progressive-disclosure discipline.

**What happens:** everything gets dumped into SKILL.md (inline code, full docs, examples) → 200–500+ lines of *always-loaded-once-triggered* context → cost goes UP, not down.

**Fix:** SKILL.md is the entry point. Extract code → `scripts/`, long docs → `references/`, templates/icons → `assets/`. Use the official scaffolders (`skill-creator`) instead of freehand generation.

---

## Why Skills > MCP

| Aspect | MCP Servers | Skills |
|--------|-------------|--------|
| **Loading** | Full tool schema upfront | Progressive disclosure |
| **Context cost** | 1,000–10,000 tokens per server | ~100 tokens (metadata) until triggered |
| **Modification** | Code changes | Edit markdown |
| **Portability** | Platform-specific protocol | Cross-vendor `SKILL.md` standard |

Skills and MCP are complementary: a skill can *declare* an MCP dependency (Codex `agents/openai.yaml`) and teach the agent *when/how* to use those tools. Use MCP for live tool access; use skills for the procedural knowledge.

**Real-world savings (this repo):** notion 9,600→600 tok (94%), telegram 3,500→400 tok (89%). Full table at the bottom.

---

## Core Principle: Progressive Disclosure (3 levels)

Every vendor implements the same three-tier lazy load. This is the entire point — "the context window is a public good."

| Level | What loads | When | Budget |
|-------|-----------|------|--------|
| **1. Discovery / metadata** | `name` + `description` only | Always, at session start | ~100 words/skill |
| **2. Activation / body** | Full `SKILL.md` | When the skill is triggered | **< 500 lines** |
| **3. Execution / resources** | `scripts/`, `references/`, `assets/` | On demand, file-by-file | effectively unlimited |

**Key insight for token economy:** a **script is executed without loading its source** into context (Level 3 with zero penalty). Prefer "run exactly this script" over pasting code the model must read. Reserve `references/` for prose the model genuinely needs to read.

**Vendor caps on the metadata tier (Level 1):**
- Claude Code: `skillListingBudgetFraction` ≈ **1%** of context window; combined `description`+`when_to_use` truncated at **1,536 chars** (`maxSkillDescriptionChars`); `/doctor` shows what got dropped.
- Codex: skills list capped at **≤2% of context, or 8,000 chars**.

---

## Universal Skill Structure (the open standard)

```
skill-name/
├── SKILL.md          # REQUIRED — the only mandatory file. Frontmatter + markdown body.
├── scripts/          # Executable code (run, not loaded). Cross-vendor.
├── references/       # Long-form docs loaded on demand. Cross-vendor.
└── assets/           # Templates, icons, fonts used in output. Cross-vendor.
```

Rules that hold everywhere:
- **Frontmatter must be the very first thing in the file** (Gemini *silently skips* the skill otherwise).
- **`name` should match the directory name**; lowercase, numbers, hyphens only, ≤64 chars.
- **Keep `references/` one level deep** — agents partially read deeply-nested files and miss content.
- **Reference files > ~100–300 lines → add a table of contents** so a `head -100` reveals scope.

> **`README.md` is NOT a skill doc channel.** The old version of this doc told you to keep docs in `README.md` for Claude/Cursor. The open standard uses **`references/`** for on-demand docs across all vendors; Codex ignores `README.md` entirely. Use `references/`.

---

## SKILL.md Template (portable)

```yaml
---
name: processing-pdfs
description: Extracts text and tables from PDF files and fills PDF forms. Use whenever the user mentions a .pdf, asks to read/merge/split/fill a PDF, even if they don't say "PDF" explicitly.
---

# Processing PDFs

One-line statement of what this does.

## Quick start
```bash
scripts/extract.py --in report.pdf --out report.md
```

## When to do what
- Read/extract → `scripts/extract.py` (see `references/extract.md`)
- Fill a flat form → `scripts/fill.py` (see `references/forms.md`)

Detailed API and edge cases: `references/`.
```

**Target:** as short as it can be while still teaching the task; hard ceiling **< 500 lines**. Push everything optional to `references/`.

---

## Writing the `description` (the single most important field)

The `description` is the **only** signal for auto-invocation, and on Gemini it's the *only* field that matters at all. Get it right:

**DO**
- Write in **third person** ("Extracts…", "Generates…") — it's injected into the system prompt; "I can help…" / "You can…" breaks discovery.
- State **what it does AND when to use it** — concrete triggers, keywords, file types. Put the **key use case first** (it gets truncated).
- Be **a little pushy** to fight under-triggering: *"Make sure to use this whenever the user mentions X, Y, Z, even if they don't explicitly ask."*
- Prefer **gerund names** (`processing-pdfs`, `analyzing-spreadsheets`), not `helper`/`utils`/`tools`.

**DON'T**
- "Helps with documents" / "Does stuff with files" — too vague to ever trigger.
- Implementation details ("uses hybrid retrieval with Cohere reranking") — describe the *job*, not the internals.

```yaml
# Good
description: Searches the personal RAG knowledge base. Use for finding documents, notes, past conversations, or transcripts — whenever the user asks "what did I save about…" or references earlier docs.
# Bad
description: RAG search using hybrid retrieval with Cohere reranking.
```

**Validation:** `name` ≤64 chars, no reserved words ("anthropic"/"claude"), no XML tags. `description` non-empty, ≤1024 chars, no XML tags.

---

## Scripts vs Inline Code

**Bad** — 50 lines of Python in SKILL.md (loaded into context every time the skill triggers).

**Good** — `scripts/send.py --target @user --text "Hello"`; full API in `references/`.

Scripts win because they (1) cost zero context until executed, (2) self-document via `--help`, (3) are reusable, (4) are more reliable for fragile/destructive tasks ("run exactly this" beats "write code like this").

**Match degrees of freedom to task fragility:** open-ended task → high-freedom prose; fragile/destructive task → exact pre-bundled script. Make execute-vs-read intent explicit.

---

## Vendor-Specific Notes

### Claude Code (reference implementation — richest)
- **Path:** `~/.claude/skills/` (personal), `.claude/skills/` (project, searched up parent dirs + monorepo nested), plugin skills namespaced `plugin:skill`. Precedence: enterprise > personal > project > bundled. Live-reloads on edit.
- **Custom commands merged into skills:** `.claude/commands/deploy.md` and `.claude/skills/deploy/SKILL.md` both yield `/deploy`. Skills are now preferred over commands.
- **Extended frontmatter** (all optional):

| Field | Meaning |
|-------|---------|
| `when_to_use` | Extra trigger phrases, appended to description |
| `argument-hint` / `arguments` | Autocomplete hint / named `$name` args |
| `disable-model-invocation` | `true` = user-only via `/name`; removes description from context |
| `user-invocable` | `false` = hidden from `/` menu (background knowledge for the model) |
| `allowed-tools` / `disallowed-tools` | Pre-approve / remove tools while skill active |
| `model` / `effort` | Override model / effort (`low`…`max`) for the turn |
| `context: fork` + `agent` | Run skill in isolated subagent (`Explore`/`Plan`/`general-purpose`) |
| `paths` | Glob — auto-activate on matching files |
| `hooks` | Hooks scoped to this skill's lifecycle |
| `shell` | `bash` (default) / `powershell` for inline `` !`cmd` `` injection |

- **Dynamic context & substitution:** `` !`cmd` `` and ` ```! ` blocks inject command output; `$ARGUMENTS`, `$N`, `$name`, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`.
- **Distribution = plugins:** add `.claude-plugin/plugin.json`; install via `/plugin install skill-creator@claude-plugins-official`.
- **Tooling:** `skill-creator` plugin ships `quick_validate.py`, `package_skill.py` (→ `.skill` file), and an eval/description-optimizer harness.
- **Bundled skills** ship in Claude Code (`/code-review`, `/debug`, `/loop`, `/run`, `/verify`…); disable via `disableBundledSkills`.

### OpenAI Codex
- **Path:** `$HOME/.agents/skills` (vendor-neutral). Discovery precedence: system < `/etc/codex/skills` < user < repo-root `.agents/skills` < cwd `.agents/skills`.
- **Frontmatter: still ONLY `name` + `description`.** Everything else → `agents/openai.yaml`:
  ```yaml
  interface: { display_name, short_description, icon_small, brand_color, default_prompt }
  policy:    { allow_implicit_invocation: false }
  dependencies:
    tools:
      - type: mcp
        value: openaiDeveloperDocs
        transport: streamable_http
        url: https://example.com/mcp
  ```
- **Docs:** `references/` only — **does not read `README.md`**.
- **Invoke:** `$skill-name` or `/skills`; implicit via `description`. Disable a skill in `~/.codex/config.toml` (`[[skills.config]] enabled = false`).
- **AGENTS.md is separate** (always-on repo instructions, `project_doc_max_bytes` default 32 KiB). Don't duplicate skill content into it.
- **New (2026):** `skill-creator` (`init_skill.py` + `quick_validate.py`), `$skill-installer` (curated catalog), `/import` from Claude Code, macOS "Record & Replay" → auto-generates a skill from a demonstrated workflow.

### Cursor (native since v2.4)
- **Path:** `~/.cursor/skills/` + `~/.agents/skills/` (user); `.cursor/skills/` + `.agents/skills/` (project). **Also auto-loads `.claude/skills/` and `.codex/skills/`** — a Claude/Codex skill works unchanged.
- **Frontmatter:** open-standard core + Cursor extensions `paths` (glob scoping) and `disable-model-invocation`.
- **Invoke:** `/skill-name` (run) or `@skill-name` (attach as context).
- **`/migrate-to-skills`:** converts dynamic rules (no globs, not always-apply) and slash commands → skills. Always-apply / glob-scoped rules stay as rules.
- **Rules vs Skills:** Rules (`.cursor/rules/*.mdc`) = always-on declarative conventions ("use TS strict mode"). Skills = on-demand procedures ("how to deploy"). Cursor effectively **deprecated dynamic rules in favor of skills**. Short always-true guidance → Rule; multi-step procedure → Skill.
- **Cloud Agents:** commit project-scoped skills to the repository. User-level folders on the local Mac are not part of a cloud checkout. `.agents/skills/` is the preferred portable project root; `.cursor/skills/` remains an explicit Cursor root.

### Gemini CLI (native, fully automatic)
- **Path:** `~/.gemini/skills/` (or `~/.agents/skills/`) user; `.gemini/skills/` (or `.agents/skills/`) workspace. Within a tier, `.agents/skills/` **wins over** `.gemini/skills/`. Tiers: built-in < extension < user < workspace.
- **Activation via `activate_skill` tool (agent-only — you cannot call it manually):** at session start Gemini injects only `name`+`description`; on a match the model calls `activate_skill`, the user gets a **confirmation prompt**, then the full SKILL.md + folder become available and the skill dir becomes an allowed file path.
- **Frontmatter:** only `name` + `description` are documented as honored. Extended fields (`allowed-tools`, `license`…) are **ignored, not errored** (community-reported, unconfirmed). Frontmatter **must be first in the file** or the skill is silently skipped.
- **Manage (not invoke):** `/skills list|enable|disable|link|reload`, `gemini skills install <URL>`. To "invoke," phrase the request to match the description, or pre-enable/disable.
- **GEMINI.md** = always-on memory (opposite token profile to skills). **Extensions** = packaging unit bundling MCP + GEMINI.md + commands + skills.
- ⚠️ **Security:** activation grants file access to the *entire* skill directory — never co-locate secrets/unrelated files in a skill folder.

---

## Cross-Platform Strategy (recommended)

Single source of truth + symlinks (this repo already does this):

```bash
mkdir -p ~/.claude/skills/my-skill           # author here (richest validator)
ln -s ~/.claude/skills/my-skill ~/.agents/skills/my-skill   # Codex + Cursor + Gemini all read .agents/skills
ln -s ~/.claude/skills/my-skill ~/.cursor/skills/my-skill   # explicit, optional (Cursor also reads .claude/)
ln -s ~/.claude/skills/my-skill ~/.gemini/skills/my-skill   # explicit, optional (Gemini also reads .agents/)
```

Because Cursor reads `.claude/`, and Codex/Cursor/Gemini-compatible hosts commonly read `.agents/`, the **single `.agents/skills/` symlink can cover several vendors**. Keep `name` + `description` portable; isolate vendor extras in their sidecars (`agents/openai.yaml`) or optional fields the others ignore. Exact Antigravity paths can vary by installation; verify the active discovery roots before publishing a workflow.

### Plugins are a separate distribution layer

Do not treat a plugin cache as another global skill root. Plugin skills are
namespaced (`plugin:skill`) and can depend on hooks, commands, MCP servers, or
platform manifests. Flattening them into `~/.agents/skills` loses that contract
and creates collisions with personal or bundled skills.

Use native plugin installation when the package supplies a harness manifest.
A managed fleet can maintain an explicit allowlist in a runner such as
`scripts/sync_agent_plugins.py` and report newly enabled Claude plugins that
have not been classified. Install each managed plugin through its native
harness mechanism; audit cloud plugin availability separately from the
physical personal-skill snapshot.

### Local vs cloud distribution

Local symlinks do not cross the machine boundary. Codex cloud and Claude Code
web start from fresh clones, so a separate private skills repository is useful
as a versioned distribution source but is **not sufficient by itself**. The
working repository must contain project-scoped skills, or its cloud environment
must install the private bundle before the agent starts.

Recommended portable repository layout:

```text
personal-agent-skills/
├── skills/<name>/...                  # physical versioned content
├── .agents/skills/<name> -> ../../skills/<name>
├── .claude/skills/<name> -> ../../skills/<name>
└── .cursor/skills/<name> -> ../../skills/<name>
```

- **Codex cloud:** the selected repository is cloned, then Codex discovers
  repo-scoped `.agents/skills/`. Environment setup scripts can install a private
  bundle into `~/.agents/skills`, but private Git authentication must be
  configured as an environment secret; never embed a token in Git or the setup
  script.
- **Claude Code web:** local `~/.claude/skills/` does not transfer. Use skills
  enabled for the claude.ai account, commit `.claude/skills/` to the working
  repository, or declare a repository plugin. Claude cloud environments may
  also fetch dependencies in a setup script.
- **Cursor Cloud Agents:** use repo-scoped `.agents/skills/` or
  `.cursor/skills/`; local user-level folders do not belong to the cloned repo.

A daily runner can publish the converged canonical set to a private snapshot
repository and block common secret formats, credential files, unsafe symlinks,
virtual environments, and caches before commit/push. Use the generated
installer to materialize the same snapshot into a target working repository.

For managed global roots, use the stateful runner documented in
`docs/DAILY_SKILL_SYNC.md`. It hashes the useful contents of the full skill
folder, replaces confirmed duplicate copies with symlinks only after backup,
propagates deletion only for a previously tracked canonical skill, and writes a
rollback manifest plus JSONL audit log. A first-seen divergent same-name pair
must remain a conflict until its canonical path is explicitly chosen; never
pick a winner from timestamps alone.

---

## Compliance Checklist

Before committing a skill:

- [ ] Frontmatter is the **first thing** in the file; `name` == directory name
- [ ] `description` is third-person, states **what + when**, includes trigger keywords, key use case first
- [ ] SKILL.md body **< 500 lines** (and as lean as the task allows)
- [ ] Code in `scripts/` (executed, not pasted); long docs in `references/` (one level deep)
- [ ] No `README.md` relied on for skill docs (Codex won't read it)
- [ ] Reference files > ~100 lines have a table of contents
- [ ] No secrets / unrelated files inside the skill folder (Gemini exposes the whole dir)
- [ ] Vendor extras isolated in sidecars (`agents/openai.yaml`), not in portable frontmatter
- [ ] Validated (`quick_validate.py` for Claude/Codex) and symlinked across vendors
- [ ] Tested with a real trigger phrase (and on a small model — descriptions that work on Opus may under-trigger on Haiku/Flash)

---

## Resources

- [Agent Skills open standard — agentskills.io](https://agentskills.io)
- [Claude Code Skills](https://code.claude.com/docs/en/skills) · [Anthropic authoring best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Codex Skills](https://developers.openai.com/codex/skills) · [Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- [Cursor Skills](https://cursor.com/docs/context/skills)
- [Codex cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment.md)
- [Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web)
- [Gemini CLI Skills](https://geminicli.com/docs/cli/skills/) · [`activate_skill`](https://geminicli.com/docs/tools/activate-skill/)

---

## Measured Results (this repo's migrations)

| Skill | Before | After | Reduction |
|-------|--------|-------|-----------|
| notion | 480 lines / ~9,600 tok | 17 lines / ~600 tok | **96%** |
| singularity | 368 lines / ~7,400 tok | 18 lines / ~600 tok | **95%** |
| playwright | 335 lines / ~6,700 tok | 22 lines / ~600 tok | **93%** |
| telegram-user | 173 lines / ~3,500 tok | 21 lines / ~400 tok | **88%** |
| youtube | 172 lines / ~3,400 tok | 17 lines / ~400 tok | **90%** |
| **TOTAL** | **2,500+ lines / ~23,000 tok** | **344 lines / ~6,900 tok** | **70%** |

> These are entry-point sizes — detail lives in `scripts/`/`references/`, loaded only on demand. The reduction comes from progressive disclosure, not from deleting knowledge.
