# Agent-Agnostic Repository Guide

A practical guide for making any repository work with any AI coding agent without vendor lock-in. Covers creating new repos agent-agnostic from day one (Part A) and transforming existing repos (Part B).

---

## The Core Principle

Every piece of agent configuration falls into exactly one of three categories:

| Category | Strategy | Examples |
|----------|----------|----------|
| **Portable** (same format across agents) | Store once in `.agents/`, sync script mirrors to agent dirs that don't read it natively | `AGENTS.md`, `SKILL.md` files |
| **Generated** (same data, different format) | Canonical source in `.agents/`, sync script renders per-agent | MCP config (JSON vs TOML) |
| **Agent-specific** (no cross-agent equivalent) | Leave in agent-native directory, referencing portable sources instead of duplicating them | `.omp/rules/*.md`, `.claude/settings.json` |

If you remember nothing else: **portable things get symlinks, generated things get a sync script, agent-specific things stay put and refer directly to portable things whenever possible.**

---

## Part A: New Repo Template

### A.1 Directory Layout

```
repo/
  AGENTS.md                           # Universal instructions (AAIF standard)
  CLAUDE.md                           # Pointer: "@AGENTS.md" + Claude-only overrides

  .agents/
    README.md                         # Self-management doc for this directory (not AGENTS.md, see A.8)
    skills/                           # Shared skills (committed)
      <skill-name>/
        SKILL.md                      # agentskills.io spec format
        references/                   # Optional supporting docs
        scripts/                      # Optional automation
    skills-local/                     # Private skills (gitignored)
      .gitkeep
    mcp/
      servers.json                    # Canonical MCP server definitions
    scripts/
      sync-skills.sh                  # Sync skills to agent-native dirs
      sync-mcp.sh                     # Sync (render) agent-native MCP configs

  .claude/                            # Generated + agent-specific
    skills/<name> -> ../../.agents/skills/<name>   # Symlinks
    settings.json                     # Agent-specific (hooks, etc.)
  .omp/                               # Generated + agent-specific (oh-my-pi)
    rules/                            # Agent-specific (omp rule frontmatter, not portable)
    mcp.json                          # Generated from .agents/mcp/servers.json
  .opencode/                          # Agent-specific (opencode)
    agents/                           # Agent-specific (subagent definitions)
  opencode.json                       # Agent-specific settings; `mcp` key generated from .agents/mcp/servers.json
  .codex/
    config.toml                       # Codex project settings; `[mcp_servers.*]` generated from .agents/mcp/servers.json
  .mcp.json                           # Generated: Claude project-level MCP
```

### A.2 Layer 1: Instructions (AGENTS.md)

**What:** Project context, conventions, routing, coding standards.
**Portable?** Yes. [AGENTS.md](https://agents.md) is the AAIF/Linux Foundation standard, read by 20+ agents including Codex, oh-my-pi (omp), Copilot, Gemini CLI, Devin, and more.
**Sync mechanism:** None needed. Most agents read it natively.

**Setup:**

1. Create `AGENTS.md` at repo root with project overview, directory structure, setup commands, coding standards, git conventions, and a pointer to `.agents/README.md`.
2. Create `CLAUDE.md` as a pointer:

```markdown
@AGENTS.md

This repo keeps reusable agent assets in `.agents/` and syncs tool-specific
adapters with `.agents/scripts/`.
```

3. For nested packages, create sub-`AGENTS.md` files (e.g., `src/package/AGENTS.md`) for directory-scoped context. Codex and omp load `AGENTS.md` files from the repo root down to the working directory. Claude Code ignores `AGENTS.md` by default once a `CLAUDE.md` exists, so put a `CLAUDE.md` containing `@AGENTS.md` next to each nested `AGENTS.md` too.

**Sources:** [AGENTS.md spec](https://github.com/agentsmd/agents.md), [AAIF / Linux Foundation](https://aaif.io/)

### A.3 Layer 2: Skills (SKILL.md)

**What:** Reusable workflows agents can invoke — `/rally`, `/learn`, `/deploy`, etc.
**Portable?** Yes. The [Agent Skills spec](https://agentskills.io/specification) defines a universal SKILL.md format adopted by 16+ agents.
**Canonical location:** `.agents/skills/<skill-name>/SKILL.md`
**Sync mechanism:** A sync script mirrors skills into agent-native dirs for agents that don't scan `.agents/skills/` themselves. Symlinks are the default strategy; copying is an equally valid implementation if an agent doesn't follow symlinks. Agents that already read `.agents/skills/` natively must **not** be sync targets: a second copy in their own config dir is at best redundant and, in Codex, shows up as a duplicate skill. (omp and opencode also scan `.claude/skills/`, but they deduplicate the symlinked copies, so Claude's sync target is harmless to them.)

| Agent | Reads `.agents/skills/` natively? | Sync target |
|-------|-----------------------------------|-------------|
| Claude Code | No (only `.claude/skills/`) | `.claude/skills/` |
| Codex | Yes (cwd up to repo root) | None |
| oh-my-pi (omp) | Yes (cwd up to repo root) | None |
| opencode | Yes (cwd up to git worktree) | None |

**SKILL.md format** ([agentskills.io spec](https://agentskills.io/specification)):

```yaml
---
name: my-skill
description: What this skill does and when to use it.
---

Step-by-step instructions for the agent...
```

Required fields: `name` (lowercase, hyphens, matches directory name), `description`.
Keep SKILL.md under 500 lines. Use `references/` for detailed docs.

**The sync script** (`.agents/scripts/sync-skills.sh`):

```bash
#!/usr/bin/env bash
# Exits 0 if nothing changed, 1 if it changed anything (so a pre-commit hook stops for
# re-staging), 2 if a real directory is in the way of a link.
set -euo pipefail

AGENTS_DIR="$(cd "$(dirname "$0")/.." && pwd)"
REPO_ROOT="$(cd "$AGENTS_DIR/.." && pwd)"
# Agents that don't discover .agents/skills/ natively
TARGETS=(".claude/skills")
# Agents that do (omp, opencode): never sync here, only clean up old links
NATIVE=(".omp/skills" ".opencode/skills")
changed=0

for target_dir in "${TARGETS[@]}"; do
    mkdir -p "$REPO_ROOT/$target_dir"

    for skill_dir in "$AGENTS_DIR"/skills/*/; do
        [ -d "$skill_dir" ] || continue
        name="$(basename "$skill_dir")"
        link_target="../../.agents/skills/$name"
        link_path="$REPO_ROOT/$target_dir/$name"

        if [ -L "$link_path" ]; then
            [ "$(readlink "$link_path")" = "$link_target" ] && continue
            rm "$link_path"
        elif [ -e "$link_path" ]; then
            echo "ERROR: $link_path exists but is not a symlink." >&2; exit 2
        fi

        ln -s "$link_target" "$link_path"
        echo "Synced: $link_path -> $link_target"; changed=1
    done

    # Prune links to skills that no longer exist (including dangling ones)
    for link in "$REPO_ROOT/$target_dir"/*; do
        [ -L "$link" ] || continue
        target="$(readlink "$link")"
        [[ "$target" == ../../.agents/skills/* ]] || continue
        [ -d "$AGENTS_DIR/skills/${target#../../.agents/skills/}" ] && continue
        rm "$link"; echo "Pruned: $link"; changed=1
    done
done

# Remove old links from agents that read .agents/skills/ natively (they duplicate skills)
for native_dir in "${NATIVE[@]}"; do
    for link in "$REPO_ROOT/$native_dir"/*; do
        [ -L "$link" ] || continue
        [[ "$(readlink "$link")" == ../../.agents/skills/* ]] || continue
        rm "$link"; echo "Pruned (reads .agents/skills/ natively): $link"; changed=1
    done
done

exit "$changed"
```

The script runs on the stock bash 3.2 that ships with macOS, and only prunes relative links into `.agents/skills/`, so hand-made links elsewhere (e.g., to `~/.agents/skills/`) are left alone.

**Installing third-party skills** ([Vercel skills CLI](https://github.com/vercel-labs/skills)):

```bash
npx skills add <repo>@<skill-name> -y
# Installs to .agents/skills/ AND auto-creates .claude/skills/ symlink
```

The Vercel CLI natively understands the `.agents/skills/` convention and creates symlinks to `.claude/skills/` automatically. The sync script is a safety net for manual skill creation.

**Private skills:** `.agents/skills-local/` is gitignored. If you extend the sync script to cover it, the synced entries are machine-local and should not be committed.

**Sources:** [Agent Skills spec](https://agentskills.io/specification), [Client implementation guide](https://agentskills.io/client-implementation/adding-skills-support.md), [npx skills](https://github.com/vercel-labs/skills), [SSW Rules: symlink agents to claude](https://www.ssw.com.au/rules/symlink-agents-to-claude)

### A.4 Layer 3: MCP Configuration

**What:** Model Context Protocol server definitions — which external tools the agent can call.
**Portable?** No. Same data, different format per agent.
**Canonical location:** `.agents/mcp/servers.json`
**Sync mechanism:** Generation script renders per-agent files.

**Why generation, not symlinks:**

| Agent | File | Format difference |
|-------|------|-------------------|
| Claude Code | `.mcp.json` | JSON. Servers nested under `mcpServers`; URL servers need `"type": "http"` |
| oh-my-pi (omp) | `.omp/mcp.json` | JSON. Servers nested under `mcpServers`, `type` is `stdio`/`http`/`sse`. Read only from the directory omp starts in |
| opencode | `opencode.json` | JSON(C). Servers nested under `mcp`; `type` is `local` (with `command` as an array) or `remote` (with `url`) |
| Codex | `.codex/config.toml` | TOML. `[mcp_servers.<name>]` tables. Loaded only for trusted projects |

**Canonical format** (`.agents/mcp/servers.json`):

```json
{
  "figma": {
    "type": "http",
    "url": "https://mcp.figma.com/mcp"
  },
  "stacks": {
    "type": "http",
    "url": "https://stacksmcp.int.stackoverflow.net/mcp"
  }
}
```

**Shared files:** `opencode.json` and `.codex/config.toml` also hold hand-written settings, so the sync script must rewrite only the MCP part (the `mcp` key, or the `[mcp_servers.*]` tables) and leave the rest untouched. If you use `opencode.jsonc`, use a JSONC-aware tool so comments survive.

**Codex trust:** Codex loads project `.codex/` layers only after you trust the project. For untrusted checkouts, the sync script can offer an `install-codex` command that merges a managed block into `~/.codex/config.toml` with sentinel comments (`# BEGIN <repo> managed MCP` / `# END <repo> managed MCP`).

**Sources:** [MCP specification](https://modelcontextprotocol.io), [AAIF](https://aaif.io/)

### A.5 Layer 4: Agent-Specific (Leave As-Is)

These have no cross-agent equivalent. Do not try to abstract them.

| File | Agent | Why not portable |
|------|-------|-----------------|
| `.omp/rules/*.md` | oh-my-pi (omp) | Rule frontmatter: on-demand rules via `description`, always-on via `alwaysApply`, stream-triggered via `condition`, optionally gated by `globs` — no equivalent elsewhere. Loaded only from the `.omp/` in the directory omp starts in (no walk-up), so launch omp from the repo root |
| `opencode.json` (all keys except `mcp`) | opencode | Models, permissions, `instructions` globs, and other opencode settings |
| `.opencode/agents/*.md` | opencode | Subagent definitions — opencode-specific feature |
| `.codex/config.toml` (all keys except `mcp_servers`) | Codex | Models, sandbox, approval, and other Codex settings |
| `.claude/settings.json` | Claude Code | Hooks, metadata updaters — Claude-specific harness features |
| `.claude/settings.local.json` | Claude Code | Machine-local permissions — always gitignored |
| `.claude/agents/*.md` | Claude Code | Subagent definitions — Claude-specific feature |

**If you want the same knowledge available in multiple agents**, put it in `AGENTS.md` (universally read) rather than trying to sync between `.omp/rules/` and `.claude/` formats.

**Agent-specific files should refer to portable sources, not copy them.** Keep the agent-specific part thin and point at the canonical content: `CLAUDE.md` imports `@AGENTS.md`, a hook in `.claude/settings.json` calls a script in `.agents/scripts/` instead of inlining its logic, and an `.omp/rules/` rule adds only the omp-specific trigger and links to the doc that holds the actual guidance.

### A.6 Automation

**Pre-commit hooks** (`.pre-commit-config.yaml`):

```yaml
repos:
  - repo: local
    hooks:
      - id: sync-agent-skills
        name: "Sync .agents/skills to agent dirs"
        entry: .agents/scripts/sync-skills.sh
        language: script
        pass_filenames: false
        always_run: true  # `files:` never matches deletions, so stale links would slip through
      - id: check-agent-mcp
        name: "Verify MCP configs are current"
        entry: .agents/scripts/sync-mcp.sh check
        language: script
        pass_filenames: false
        files: ^\.agents/mcp/
```

When `sync-skills.sh` changes anything it exits non-zero, so the commit stops; stage the synced entries and commit again. Because the hook always runs, an uncommitted skill directory in `.agents/skills/` also blocks unrelated commits: stage the skill together with its link, or keep drafts in `.agents/skills-local/`.

### A.7 .gitignore

```gitignore
# Agent skills (machine-local)
.agents/skills-local/*
!.agents/skills-local/.gitkeep

# Agent settings (machine-local)
.claude/settings.local.json
.claude/worktrees/
```

For **whitelist repos** (meta workspace pattern where `*` ignores everything), put the allow-list first and repeat the machine-local ignores after it. Git uses the last matching pattern, so `!.agents/**` and `!.claude/**` would otherwise re-include private files:

```gitignore
!.agents/
!.agents/**
!.claude/
!.claude/**
!.codex/
!.codex/**
!.omp/
!.omp/**
!.opencode/
!.opencode/**
!opencode.json
!opencode.jsonc
!.mcp.json
!AGENTS.md
!CLAUDE.md
!.gitignore
!.pre-commit-config.yaml

# Machine-local, must come after the allow-list
.agents/skills-local/*
!.agents/skills-local/.gitkeep
.claude/settings.local.json
.claude/worktrees/
```

### A.8 `.agents/README.md` Template

This is the self-management doc that lets any agent maintain the directory. Name it `README.md`, not `AGENTS.md`: omp loads `.agents/AGENTS.md` as project context at the same level as the repo root, where it outranks and hides the root `AGENTS.md`. Point to it from the root `AGENTS.md` instead.

```markdown
# .agents/

Agent-agnostic configuration following the [Agent Skills](https://agentskills.io)
open standard and [AGENTS.md](https://agents.md) conventions.

## Layout

- `skills/` — Shared skills (committed). Each is a directory with `SKILL.md`.
- `skills-local/` — Private skills (gitignored).
- `mcp/servers.json` — Canonical MCP server config.
- `scripts/` — Sync and validation scripts.

## Adding a skill

1. Create `.agents/skills/<name>/SKILL.md` with `name` + `description` frontmatter
2. Run `.agents/scripts/sync-skills.sh`
3. Commit the skill directory and the synced entries

## Installing third-party skills

    npx skills add <repo>@<skill> -y

## Adding an MCP server

1. Edit `.agents/mcp/servers.json`
2. Run `.agents/scripts/sync-mcp.sh`
3. Commit all generated files

## Rules

- Edit skills in `.agents/skills/`, never in `.claude/skills/`
- Edit MCP in `.agents/mcp/servers.json`, never in `.mcp.json`, `.omp/mcp.json`, or the `mcp` key of `opencode.json`
- The pre-commit hook re-syncs skills and stops the commit until the synced entries are staged
```

### A.9 New Repo Checklist

1. Create `AGENTS.md` at repo root
2. Create `CLAUDE.md` with `@AGENTS.md` pointer
3. Create `.agents/` with `skills/`, `skills-local/.gitkeep`, `mcp/servers.json`, `scripts/`, `README.md`
4. Create initial skills in `.agents/skills/`
5. Run `sync-skills.sh` to sync skills to agent dirs
6. Create or adapt sync script for MCP
7. Run sync to generate `.mcp.json`, `.omp/mcp.json`, the `mcp` key of `opencode.json`, `.codex/config.toml`
8. Add pre-commit hooks
9. Update `.gitignore`
10. Verify: synced skills resolve, sync check passes, skills discoverable

---

## Part B: Migration Guide

### B.1 Assessment: Audit What You Have

```bash
# Find all agent config files
find . -maxdepth 4 \( \
  -name "AGENTS.md" -o -name "CLAUDE.md" -o \
  -name ".mcp.json" -o -name "mcp.json" -o \
  -path "*/rules/*.md" -o -path "*/agents/*.md" -o -name "SKILL.md" -o \
  -name "settings.json" -o -name "settings.local.json" -o \
  -name "config.toml" -o -name "opencode.json*" \
\) -not -path "*/.venv/*" -not -path "*/node_modules/*"
```

For each file, classify as Portable, Generated, or Agent-specific.

### B.2 Classification Matrix

| Asset | Category | Action |
|-------|----------|--------|
| `AGENTS.md` | Portable | Keep at root. Create if missing. |
| `CLAUDE.md` (full instructions) | Portable | Move content to `AGENTS.md`, replace with `@AGENTS.md` pointer |
| `.claude/skills/*/SKILL.md`, `.omp/skills/*/SKILL.md`, `.opencode/skills/*/SKILL.md` (real files) | Should be portable | `git mv` to `.agents/skills/`, then run `sync-skills.sh` |
| `.agents/skills/*/SKILL.md` | Already portable | No change needed |
| `.omp/rules/*.md` | Agent-specific | **Leave in place.** omp rule frontmatter not portable. |
| `.mcp.json` | Generated | Create `.agents/mcp/servers.json` as canonical, generate from it |
| `.omp/mcp.json` | Generated | Generate from canonical source |
| `opencode.json` (`mcp` key) | Generated | Generate from canonical source; leave other keys alone |
| `opencode.json` (other keys), `.opencode/agents/` | Agent-specific | Leave |
| `.codex/config.toml` (`[mcp_servers.*]`) | Generated | Generate from canonical source; leave other tables alone |
| `.claude/settings.json` | Agent-specific | Leave (committed hooks) |
| `.claude/settings.local.json` | Agent-specific | Leave (gitignored permissions) |

### B.3 Migration Steps by Layer

**Order matters.** Start with the lowest-risk layer:

#### Step 1: Instructions (lowest risk)

- If `AGENTS.md` exists: review, keep as canonical
- If only `CLAUDE.md` exists with full instructions: rename to `AGENTS.md`, create pointer `CLAUDE.md`
- If both exist with different content: merge into `AGENTS.md`, reduce `CLAUDE.md` to pointer + overrides

#### Step 2: Skills

- Inventory: check `.claude/skills/`, `.omp/skills/`, `.opencode/skills/`, `.agents/skills/`
- If real files in `.claude/skills/`, `.omp/skills/`, or `.opencode/skills/`: `git mv` to `.agents/skills/`, then run `sync-skills.sh`
- If already in `.agents/skills/`: add `sync-skills.sh` and run it
- Add pre-commit hook

#### Step 3: MCP

- Find the most complete MCP config (usually `.mcp.json`)
- Copy to `.agents/mcp/servers.json` as canonical
- Create or adapt sync script
- Run sync, diff outputs against originals to verify
- Add to pre-commit or CI

#### Step 4: Cleanup

- Add `.agents/README.md` (rename an existing `.agents/AGENTS.md`, which shadows the root `AGENTS.md` in omp)
- Update `.gitignore`
- Verify with sync check

### B.4 Archetype-Specific Guidance

#### Archetype 1: Code Project (already partially migrated)

**Pattern:** meta-stack, sofa — Python/UV repos with some `.agents/` structure already.

**Typical state:** Has `.agents/skills/`, maybe an ad-hoc script that copies skills around, AGENTS.md at root.

**Migration:**

- Replace ad-hoc skill copying with `sync-skills.sh` (symlinks by default, copies only where an agent needs them)
- Add the pre-commit hook
- Ensure MCP generation covers all agents
- Single atomic commit

**Reference implementation:** [sofa PR #94](https://github.com/StackEng/sofa/pull/94) demonstrates the full migration.

#### Archetype 2: Agent-Specific Rules with Submodules

**Pattern:** a meta repo with many `.omp/rules/` files and git submodules, each with its own agent config.

**Key decisions:**

1. **Do NOT migrate omp rules.** The `.omp/rules/*.md` files use omp-specific features (`description` for on-demand loading, `alwaysApply`, stream-triggered `condition`s gated by `globs`) that have no cross-agent equivalent. Leave them in `.omp/rules/`. If you want Claude Code to see the same knowledge, put it in `AGENTS.md` and have the rule refer to it — don't create a new abstraction layer.

2. **Each submodule manages its own config.** Don't centralize submodule agent config in the parent repo. Apply the migration independently to each submodule that needs it. The parent's `.omp/rules/` handles meta-level navigation only.

3. **MCP across submodules.** If submodules need different MCP servers, each gets its own `.agents/mcp/servers.json`. If they share the same set, the parent's config applies in the multi-root workspace context.

#### Archetype 3: Meta Workspace / Coordination Hub

**Pattern:** iCloud Documents — portfolio coordination with 27+ sub-projects, not a code project.

**Key decisions:**

1. **Whitelist `.gitignore`** needs explicit allows for every agent directory. When a new agent tool appears, add it to the whitelist.

2. **Skills are operational workflows**, not code-generation — `/rally` triages tickets, `/route` dispatches to sub-projects. The SKILL.md format works identically for these.

3. **`AGENTS.md` is a router**, not a code guide:

```markdown
## Routing
- **gear tools**: See `gear/AGENTS.md`
- **Jira tasks**: Read `sherpa/guides/career-sherpa/gear/jira-ticket-template.md`
```

4. **MCP servers are workspace-wide.** Individual sub-projects have their own repos with their own MCP configs.

### B.5 Decision Tree

```
Is this file read identically by multiple agents?
  YES -> Portable. Move to .agents/ or root, sync to agent dirs that need it.

  NO -> Is the underlying data the same, just different format?
    YES -> Generated. Canonical source in .agents/, sync script renders.

    NO -> Agent-specific. Leave where it is, referring to portable sources where possible.
```

### B.6 Ongoing Maintenance

| Task | How |
|------|-----|
| Add a skill | Create in `.agents/skills/`, run `sync-skills.sh`, commit both |
| Install third-party skill | `npx skills add <repo>@<name> -y` |
| Add MCP server | Edit `.agents/mcp/servers.json`, run sync, commit all generated files |
| Add new agent tool | Add rendering to `sync-mcp.sh`; add a target to the `sync-skills.sh` TARGETS array unless the agent reads `.agents/skills/` natively |
| Validate skills | `skills-ref validate .agents/skills/<name>` |
| Check sync freshness | `sync-mcp.sh check` (exits non-zero if stale) |

---

## Sources

| Standard | What it governs | Link |
|----------|----------------|------|
| Agent Skills spec | SKILL.md format, skill discovery | [agentskills.io/specification](https://agentskills.io/specification) |
| AGENTS.md / AAIF | Universal agent instructions | [agents.md](https://agents.md), [AAIF (Linux Foundation)](https://aaif.io/) |
| Model Context Protocol | MCP server definitions | [modelcontextprotocol.io](https://modelcontextprotocol.io) |
| Vercel Skills CLI | `npx skills add`, skill marketplace | [github.com/vercel-labs/skills](https://github.com/vercel-labs/skills) |
| skills-ref | Skill validation CLI | [github.com/agentskills/agentskills](https://github.com/agentskills/agentskills) |
| SSW Rules | Symlink best practices | [ssw.com.au/rules/symlink-agents-to-claude](https://www.ssw.com.au/rules/symlink-agents-to-claude) |
| Claude Code Skills Docs | Claude-specific skill discovery | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) |
| Codex docs | Codex skills and project config | [learn.chatgpt.com/docs/build-skills](https://learn.chatgpt.com/docs/build-skills), [config basics](https://learn.chatgpt.com/docs/config-file/config-basic) |
| opencode docs | opencode config, skill, and MCP discovery | [opencode.ai/docs](https://opencode.ai/docs/config/) |
| oh-my-pi docs | omp skill, rule, MCP, and context-file discovery | [github.com/can1357/oh-my-pi/tree/main/docs](https://github.com/can1357/oh-my-pi/tree/main/docs) |
| Client Implementation Guide | How agents should scan `.agents/skills/` | [agentskills.io/client-implementation](https://agentskills.io/client-implementation/adding-skills-support.md) |
| Claude Code .agents/ support request | Community request for native scanning | [github.com/anthropics/claude-code#31005](https://github.com/anthropics/claude-code/issues/31005) |
