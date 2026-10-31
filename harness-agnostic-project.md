# Harness-Agnostic Project Blueprint

Share project instructions, skills, and tool configuration across AI coding
harnesses through small, explicit adapters. Part A covers new repositories;
Part B covers migration.

This standalone foundation defines harness integration for projects in any
language and for coordination hubs.

## Scope and Terminology

A **harness** surrounds a model with software and configuration for loading instructions, exposing tools, managing context, and enforcing execution policies. An **agent** combines a model and a harness to perform tasks; one harness implementation can support multiple agents or subagents with different roles. [LangChain's model-plus-harness definition](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness) includes instructions, tools, and skills in the harness.

This blueprint distinguishes **harness implementations** (Claude Code, Codex CLI, oh-my-pi, opencode) from the **project inputs** they load (instructions, skills, MCP definitions). "Across harnesses" means across implementations; "harness-specific" means tied to one implementation. Below, **Codex** means Codex CLI and **omp** means oh-my-pi.

**Harness-agnostic** means shared guidance and workflows are independent of any single implementation. Supported harnesses consume canonical sources through native discovery, imports, links, settings that point to shared files, or generated configuration. Adapters and validation are still required; behavior can differ across harnesses and models.

Keep the established names `AGENTS.md`, `.agents/`, and **Agent Skills**: they identify files, directories, and a specification shared across harnesses.

Record supported harnesses and tested versions in `.agents/README.md` (A.8). Validate instruction loading, skill discovery, and MCP configuration in each supported version. Check version-dependent examples against that harness's documentation.

---

## The Core Principle

Classify configuration items by how they can be shared. One file may contain generated and harness-specific sections.

| Category | Strategy | Examples |
|----------|----------|----------|
| **Portable** (shared format and compatible meaning) | Store once at the root or in `.agents/`; use native discovery, a symlink, an import (`CLAUDE.md` → `@AGENTS.md`), or a setting that points to it | `AGENTS.md`, portable `SKILL.md` skills |
| **Generated** (shared data, different formats) | Store canonical data in `.agents/`; render each supported harness's configuration | Shared MCP server definitions |
| **Harness-specific** (no portable mapping) | Keep in the harness's configuration; refer to shared sources instead of copying them | `.omp/rules/*.md`, `.claude/settings.json` |

---

## Part A: New Project Repository Template

### A.1 Directory Layout

```
repo/
  AGENTS.md                  # Shared instructions
  CLAUDE.md                  # "@AGENTS.md" + Claude Code-specific overrides
  .agents/
    README.md                # How to maintain this directory (not AGENTS.md, see A.8)
    skills/<name>/SKILL.md   # Shared skills, with optional references/ and scripts/
    skills-local/.gitkeep    # Private skills (gitignored)
    mcp/servers.json         # Canonical MCP servers
    scripts/
      sync-skills.sh         # Links skills into harness directories
      sync-mcp.sh            # Renders harness MCP configs
  .claude/
    skills/<name> -> ../../.agents/skills/<name>
    settings.json            # Harness-specific
  .omp/                      # oh-my-pi
    rules/                   # Harness-specific
    mcp.json                 # Generated
  .opencode/agents/          # Harness-specific
  opencode.json              # Harness-specific; `mcp` key generated
  .codex/config.toml         # Harness-specific; `[mcp_servers.*]` tables generated
  .mcp.json                  # Generated (Claude Code)
  .gemini/settings.json      # Optional: makes Gemini CLI read AGENTS.md
```

### A.2 Instructions (AGENTS.md)

[AGENTS.md](https://agents.md), the AAIF/Linux Foundation standard, holds the project overview, layout, setup commands, coding standards, git conventions, and a link to `.agents/README.md`. Codex, omp, Copilot, Devin, and other harnesses read it natively. Adapter examples:

- **Claude Code** reads `CLAUDE.md`. Import `AGENTS.md`, then add any Claude Code-specific overrides:

  ```markdown
  @AGENTS.md
  ```

- **Gemini CLI** reads `GEMINI.md` unless `.gemini/settings.json` says otherwise: `{"context": {"fileName": ["AGENTS.md", "GEMINI.md"]}}`.

The Gemini example covers instructions only. Full support also requires skill discovery and MCP adapters (B.6), not provided here.

For nested packages, add files such as `src/package/AGENTS.md`. Codex and omp load each `AGENTS.md` from the repo root to the working directory. Claude Code ignores `AGENTS.md` once `CLAUDE.md` exists, so place a `CLAUDE.md` containing `@AGENTS.md` beside each nested one.

### A.3 Skills (SKILL.md)

Skills package reusable instructions and workflows for agents. The [Agent Skills spec](https://agentskills.io/specification) defines a shared format:

```yaml
---
name: my-skill          # Lowercase and hyphens, matches the directory name
description: What this skill does and when to use it.
---

Step-by-step instructions for the agent...
```

Keep `SKILL.md` under 500 lines and move details to `references/`.

A shared format does not guarantee compatible behavior. Claude Code's `context: fork` and inline shell-command substitution are not portable by default; the spec's experimental `allowed-tools` field also has varying support. Use only instructions, resources, and features compatible with all supported harnesses in shared skills. Keep harness-specific skills separate, subject to the discovery rules below.

**Discovery.** Link skills only into harnesses that do not scan `.agents/skills/`; extra links are redundant and Codex lists them as duplicates.

| Harness | Reads `.agents/skills/`? | Sync target |
|-------|--------------------------|-------------|
| Claude Code | No | `.claude/skills/` |
| Codex | Yes (cwd up to repo root) | None |
| omp | Yes (cwd up to repo root) | None |
| opencode | Yes (cwd up to git worktree) | None |

omp and opencode also scan `.claude/skills/`; verify that your versions deduplicate the links.

Harness-specific directories do not ensure isolation. For incompatible skills, use a location other supported harnesses do not scan. If none exists, make the skill portable or keep it in its harness-specific directory and record unsupported harness/skill combinations in `.agents/README.md`. Recording a limitation does not resolve incompatibility.

**The sync script** (`.agents/scripts/sync-skills.sh`):

```bash
#!/usr/bin/env bash
# Exits 0 if the links are in sync and none are missing from the commit, 1 if it changed
# anything or a link or new skill is untracked or unstaged (so a pre-commit hook stops),
# 2 if something it doesn't manage is in the way, or git fails.
set -euo pipefail

AGENTS_DIR="$(cd "$(dirname "$0")/.." && pwd)"
REPO_ROOT="$(cd "$AGENTS_DIR/.." && pwd)"
MANAGED="../../.agents/skills"
# Harnesses that don't discover .agents/skills/ natively
TARGETS=(".claude/skills")
# Harnesses that do (omp, opencode) and leftovers from older setups: never sync here, only
# clean up old links
NATIVE=(".omp/skills" ".opencode/skills" ".codex/skills")
changed=0

# The script owns only links of the exact form it creates: $MANAGED/<one skill name>
is_managed() {
    local name="${1#"$MANAGED"/}"
    [ "$name" != "$1" ] && [ -n "$name" ] && [[ "$name" != */* ]] &&
        [ "$name" != "." ] && [ "$name" != ".." ]
}

for target_dir in "${TARGETS[@]}"; do
    if [ -L "$REPO_ROOT/$target_dir" ]; then
        # Linking the whole dir (e.g., .claude/skills -> ../.agents/skills) already covers it
        [ "$(readlink "$REPO_ROOT/$target_dir")" = "../.agents/skills" ] && continue
        echo "ERROR: $target_dir is a symlink to somewhere other than .agents/skills." >&2; exit 2
    fi
    mkdir -p "$REPO_ROOT/$target_dir"

    for skill_dir in "$AGENTS_DIR"/skills/*/; do
        [ -d "$skill_dir" ] || continue
        name="$(basename "$skill_dir")"
        link_target="$MANAGED/$name"
        link_path="$REPO_ROOT/$target_dir/$name"

        if [ -L "$link_path" ]; then
            current="$(readlink "$link_path")"
            [ "$current" = "$link_target" ] && continue
            if ! is_managed "$current"; then
                echo "ERROR: $link_path is a hand-made link to $current; rename one of the skills." >&2; exit 2
            fi
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
        is_managed "$target" || continue
        [ -d "$AGENTS_DIR/skills/${target#"$MANAGED"/}" ] && continue
        rm "$link"; echo "Pruned: $link"; changed=1
    done
done

# Remove old links from harnesses that read .agents/skills/ natively (they duplicate skills)
for native_dir in "${NATIVE[@]}"; do
    if [ -L "$REPO_ROOT/$native_dir" ]; then
        # Never prune through a dir alias: it could point at .claude/skills or .agents/skills
        echo "ERROR: $native_dir is a symlink; this harness reads .agents/skills/ natively, so remove it." >&2; exit 2
    fi
    for link in "$REPO_ROOT/$native_dir"/*; do
        [ -L "$link" ] || continue
        is_managed "$(readlink "$link")" || continue
        rm "$link"; echo "Pruned (reads .agents/skills/ natively): $link"; changed=1
    done
done

# Fail while a link or a new skill would be left out of the commit
if git -C "$REPO_ROOT" rev-parse --git-dir >/dev/null 2>&1; then
    links="$(git -C "$REPO_ROOT" status --porcelain --untracked-files=all -- "${TARGETS[@]}" "${NATIVE[@]}")" ||
        { echo "ERROR: git status failed." >&2; exit 2; }
    skills="$(git -C "$REPO_ROOT" status --porcelain --untracked-files=all -- .agents/skills)" ||
        { echo "ERROR: git status failed." >&2; exit 2; }
    # Second status column set: not staged (or untracked) yet
    pending="$({ printf '%s\n' "$links" | grep '^.[^ ]'; printf '%s\n' "$skills" | grep '^??'; } || true)"
    if [ -n "$pending" ]; then
        echo "Not staged yet (stage them, or move draft skills to .agents/skills-local/):" >&2
        echo "$pending" >&2; changed=1
    fi
fi

exit "$changed"
```

- Runs on macOS's bash 3.2.
- Manages only links of the form `../../.agents/skills/<name>`; conflicts stop it with exit 2. Exclude hand-made, machine-local links (e.g., to `~/.agents/skills/`) through `.git/info/exclude` so staging checks ignore them.
- Handles the adjacent `.agents/skills/`. Package-scoped skills need that package's own directory, script, and hook entry; root skills are visible repo-wide.
- Creates symlinks only. On Windows, use Git Bash or WSL and enable `core.symlinks` plus Developer Mode (or admin rights) before cloning. Otherwise links check out as text files and trigger exit 2. Where symlinks are unavailable, implement a copy mode that tracks owned copies and removes stale ones.

**Third-party skills** ([Vercel skills CLI](https://github.com/vercel-labs/skills)):

```bash
npx skills add <repo>@<skill-name> -y
.agents/scripts/sync-skills.sh
```

Always run the sync script after installation: the CLI selects harness directories by detection. If it copied a skill into `.claude/skills/`, sync stops. Verify it matches (`diff -r .claude/skills/<name> .agents/skills/<name>`), delete the copy, and rerun.

**Private skills** go in `.agents/skills-local/` (gitignored). If you extend the script to link them, don't commit those links.

### A.4 MCP Configuration

Store shared MCP server definitions in `.agents/mcp/servers.json` and generate each harness's format:

| Harness | File | Format |
|-------|------|--------|
| Claude Code | `.mcp.json` | JSON, servers under `mcpServers`; URL servers need `"type": "http"` |
| omp | `.omp/mcp.json` | JSON, servers under `mcpServers`; `type` is `stdio`/`http`/`sse`. Read only from the directory omp starts in |
| opencode | `opencode.json` | JSON(C), servers under `mcp`; `type` is `local` (`command` as an array) or `remote` (`url`) |
| Codex | `.codex/config.toml` | TOML, `[mcp_servers.<name>]` tables. Loaded only for trusted projects |

**Canonical format:**

```json
{
  "figma": {
    "type": "http",
    "url": "https://mcp.figma.com/mcp"
  },
  "my-tool": {
    "type": "stdio",
    "command": "uvx",
    "args": ["my-mcp-server"],
    "env": { "MY_TOOL_API_KEY": "${MY_TOOL_API_KEY}" }
  }
}
```

Use a bare map of server names, without a `mcpServers` wrapper. Servers are `stdio` (`command`, `args`, `env`) or `http` (`url`, `headers`). Store no secrets: use `${NAME}` as a whole `env` or header value, or `Bearer ${NAME}` in an `Authorization` header.

**Implement `sync-mcp.sh`** (not included), typically in Python or `jq`. It should:

- render each server for each supported harness based on its `type`:
  - Claude Code: canonical fields under `mcpServers`; placeholders work as-is
  - opencode: `local`/`remote`; `command` and `args` merged into one `command` array; `environment` instead of `env`; `{env:NAME}` instead of `${NAME}`
  - Codex: an `env` entry `${NAME}` with the same name becomes `env_vars = ["NAME"]`; `Bearer ${NAME}` becomes `bearer_token_env_var = "NAME"`; other placeholder headers go in `env_http_headers`, static ones in `http_headers`
  - omp: check its docs for placeholder support
- never expand placeholders into values
- in `opencode.json` and `.codex/config.toml`, rewrite only the `mcp` key or the `[mcp_servers.*]` tables (with a JSONC-aware tool for `opencode.jsonc`), and remove only servers it generated
- write files atomically
- support `check`: render to a temporary location, compare, exit non-zero on differences, change nothing
- fail on a field or placeholder it can't map, instead of dropping it

For definitions outside this format (SSE, OAuth, harness-specific options), extend the format and generator or configure them per harness. Track generated server names so hand-maintained entries remain untouched.

**Codex trust:** Codex loads `.codex/` only in trusted projects; review the checkout, then trust it. Writing to `~/.codex/config.toml` affects every project and risks name collisions. Offer that only as an explicit opt-in, such as `install-codex`, with a marked block (`# BEGIN <repo> managed MCP` / `# END <repo> managed MCP`), collision refusal, and matching uninstall.

### A.5 Harness-Specific Files (Leave As-Is)

| File | Harness | Contents |
|------|-------|----------|
| `.omp/rules/*.md` | omp | Rules triggered by `description`, `alwaysApply`, `condition`, `globs`. Loaded only from the directory omp starts in, so launch omp from the repo root |
| `opencode.json` (except `mcp`), `.opencode/agents/*.md` | opencode | Settings, subagents |
| `.codex/config.toml` (except `mcp_servers`) | Codex | Models, sandbox, approvals |
| `.claude/settings.json`, `.claude/agents/*.md` | Claude Code | Hooks, subagents |
| `.claude/settings.local.json` | Claude Code | Machine-local permissions (gitignored) |

Do not sync between these formats. Keep shared guidance in `AGENTS.md` and harness-specific files thin: an `.omp/rules/` rule adds only its trigger and a link to shared guidance; a Claude Code hook calls a script in `.agents/scripts/`.

### A.6 Automation

**Pre-commit hooks** (`.pre-commit-config.yaml`):

```yaml
repos:
  - repo: local
    hooks:
      - id: sync-agent-skills
        name: "Sync .agents/skills to harness directories"
        entry: .agents/scripts/sync-skills.sh
        language: script
        pass_filenames: false
        always_run: true  # `files:` never matches deletions, so stale links would slip through
      - id: check-agent-mcp
        name: "Verify MCP configs are current"
        entry: .agents/scripts/sync-mcp.sh check
        language: script
        pass_filenames: false
        always_run: true  # Also catches hand edits to generated files and generator changes
```

Stage every new skill and managed link. Sync changes, untracked or unstaged links, and untracked new skills stop the commit; fix staging and retry. Keep drafts in `.agents/skills-local/` so untracked skills do not block every commit.

During `git commit`, pre-commit sets aside unstaged tracked edits, so checks see the staged links. `pre-commit run --all-files` also sees unstaged edits.

Hooks run only after `pre-commit install` in each clone, and `--no-verify` skips them. Also run `pre-commit run --all-files` in CI.

### A.7 .gitignore

```gitignore
.agents/skills-local/*
!.agents/skills-local/.gitkeep
.claude/settings.local.json
.claude/worktrees/
```

In a repo where `*` ignores everything, put allow-list entries before machine-local ignores. Git's last matching pattern wins; reversing the order would re-include private files:

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
!.gemini/
!.gemini/settings.json
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

Link this maintenance guide from root `AGENTS.md`. Name it `README.md`: omp loads `.agents/AGENTS.md` at the root's level, hiding the root `AGENTS.md`.

```markdown
# .agents/

Harness-agnostic configuration following the [Agent Skills](https://agentskills.io)
and [AGENTS.md](https://agents.md) conventions.

## Supported Harnesses

Record tested support below. When adding or upgrading a harness, verify instruction
loading, duplicate-free skill discovery, and MCP server discovery and connectivity.
Record adapters, limitations, and unsupported harness/skill combinations.

| Harness | Tested version | Validated on | Adapters | Limitations |
|---------|----------------|--------------|----------|-------------|
| <name> | <version> | <YYYY-MM-DD> | <required adapters> | <known limitations or none> |

## Layout

- `skills/` — Shared skills (committed), one directory with a `SKILL.md` each
- `skills-local/` — Private skills (gitignored)
- `mcp/servers.json` — Canonical MCP servers
- `scripts/` — Sync scripts

## Adding a skill

1. Create `.agents/skills/<name>/SKILL.md` with `name` and `description` frontmatter
   (or `npx skills add <repo>@<skill> -y`)
2. Run `.agents/scripts/sync-skills.sh`
3. Commit the skill and its links

## Adding an MCP server

1. Edit `.agents/mcp/servers.json`
2. Run `.agents/scripts/sync-mcp.sh`
3. Commit all generated files

## Rules

- Edit shared skills here, not through `.claude/skills/` links.
- For harness-specific skills (A.3), use a location incompatible harnesses do not
  scan, make the skill portable, or keep its harness-specific location and record
  unsupported combinations under Supported Harnesses.
- Edit MCP servers only in `mcp/servers.json`; the harness MCP configs are generated
- The pre-commit hook syncs skills and blocks commits missing links or new skills
```

### A.9 New Project Repository Checklist

1. Create `AGENTS.md` and the adapters your supported harnesses need, such as `CLAUDE.md` (`@AGENTS.md`) for Claude Code or `.gemini/settings.json` for Gemini CLI
2. Create `.agents/` (A.1) and its support record in `README.md` (A.8)
3. Add `sync-skills.sh`, write `sync-mcp.sh` (A.4), and `chmod +x .agents/scripts/*.sh`
4. Add skills and run `sync-skills.sh`. It exits 1 after creating links and until they are staged, so don't chain it with `&&`
5. Run `sync-mcp.sh`
6. Update `.gitignore` (A.7)
7. Add `.pre-commit-config.yaml`, install pre-commit (e.g., `uv tool install pre-commit`), and run `pre-commit install`; every clone needs this
8. Stage everything, run `pre-commit run --all-files`, and validate each harness as described in A.8

---

## Part B: Migration Guide

### B.1 Assessment

```bash
# Harness config files, at any depth
find . \( -name .git -o -name .venv -o -name node_modules \) -prune -o \( \
  -name "AGENTS.md" -o -name "CLAUDE.md" -o -name "GEMINI.md" -o \
  -name ".mcp.json" -o -name "mcp.json" -o \
  -path "*/rules/*.md" -o -path "*/agents/*.md" -o -name "SKILL.md" -o \
  -name "settings.json" -o -name "settings.local.json" -o \
  -name "config.toml" -o -name "opencode.json*" \
\) -print

# Symlinked skills (the search above doesn't follow them)
find . \( -name .git -o -name .venv -o -name node_modules \) -prune -o \
  -type l -path "*skills*" -exec ls -l {} +
```

### B.2 Classification Matrix

| Asset | Category | Action |
|-------|----------|--------|
| `AGENTS.md` | Portable | Keep at root; create if missing |
| `CLAUDE.md` with full instructions | Portable + harness-specific | Move shared guidance to `AGENTS.md`; retain its import and Claude Code-specific overrides in `CLAUDE.md` |
| Real skill dirs under `.claude/skills/`, `.omp/skills/`, `.opencode/skills/`, `.codex/skills/` | Portable or harness-specific | Move portable skills to `.agents/skills/`; apply A.3's discovery rules and fallback to harness-specific skills |
| `.agents/skills/*/` | Portable | None |
| `.mcp.json`, `.omp/mcp.json`, `opencode.json` `mcp` key, `.codex/config.toml` `[mcp_servers.*]` | Generated | Merge into `.agents/mcp/servers.json`, then generate |
| `.omp/rules/`, `.claude/settings*.json`, other `opencode.json` and `.codex/config.toml` settings, `.opencode/agents/` | Harness-specific | Leave |

### B.3 Migration Steps by Layer

Go from lowest to highest risk:

1. **Instructions.** Make `AGENTS.md` canonical: keep it if present, create it otherwise. Move or merge shared guidance from `CLAUDE.md`, leaving only the import and Claude Code-specific overrides there.
2. **Skills.**
   - Check every skills directory from B.2, including in nested packages.
   - Apply A.3's discovery rules and fallback to harness-specific skills.
   - Merge same-named skills with different content by hand.
   - `git mv` each portable skill's whole directory, including resources and scripts, to `.agents/skills/` at the same root or package scope (A.3).
   - Run `sync-skills.sh` and add the hook.
3. **MCP.**
   - Collect server definitions from every supported harness.
   - Convert them to the canonical format (A.4): take the entries without the wrapper (e.g., `jq .mcpServers .mcp.json`), translate harness-specific fields back (opencode `local` with a command array becomes `stdio` with `command` and `args`), and replace literal secrets with placeholders.
   - Resolve conflicting same-named servers by hand.
   - Implement and run `sync-mcp.sh`; diff against the originals and verify every change is intentional.
   - Add the check to pre-commit and CI.
4. **Cleanup.** Add `.agents/README.md` (rename `.agents/AGENTS.md` if present; see A.8), update `.gitignore`, and run `pre-commit run --all-files`.

### B.4 Archetype-Specific Guidance

**Partially migrated project** (already has root `AGENTS.md` and `.agents/skills/`, possibly with a skill-copy script): replace copy scripts with `sync-skills.sh` (symlinks only, A.3), add hooks, cover every supported harness in MCP generation, and land the migration in one commit.

**Repo with submodules and harness-specific rules:**

- Leave `.omp/rules/` as is (A.5); the parent's rules handle only meta-level navigation.
- Migrate each submodule on its own instead of centralizing its config in the parent.
- MCP discovery starts from the session's launch directory, not the edited file: omp reads only that directory; opencode searches up to the Git boundary. A parent-root session uses the parent's servers even when editing a submodule; a submodule-root session uses the submodule's (for omp, start at that exact root). User-level servers apply in both cases. Give submodules their own `.agents/mcp/servers.json` when people start sessions there.

**Coordination hub** (not a code project; many sub-projects in their own repos):

- Allow each supported harness directory in an allow-list `.gitignore` (A.7); add entries when adopting new harnesses.
- Skills are operational workflows (triage tickets, dispatch work to sub-projects); the format is the same.
- `AGENTS.md` is a router rather than a code guide:

  ```markdown
  ## Routing
  - **Billing service**: see `billing/AGENTS.md`
  - **Release notes**: follow `docs/templates/release-notes.md`
  ```

- The hub's MCP servers apply to sessions started at the hub root; sub-projects have their own.

### B.5 Decision Tree

```
Does this item have the same format and compatible meaning across supported harnesses,
using native discovery, links, imports, or settings that point to shared files?
  YES -> Portable: one source at the root or in .agents/; adapt discovery.
  NO  -> Can a generator translate shared data into each supported harness's format without losing meaning?
    YES -> Generated: canonical source in .agents/, rendered by a script.
    NO  -> Harness-specific: leave it, referring to portable sources where possible.
```

### B.6 Ongoing Maintenance

| Task | How |
|------|-----|
| Add or install a skill | Create it in `.agents/skills/` or `npx skills add`, run `sync-skills.sh`, commit skill and links |
| Add an MCP server | Edit `.agents/mcp/servers.json`, run `sync-mcp.sh`, commit generated files |
| Support a new harness | Add any needed instruction adapter and MCP rendering; add it to `TARGETS` in `sync-skills.sh` unless it reads `.agents/skills/` natively; validate and record support in `.agents/README.md` |
| Upgrade a supported harness | Recheck instruction loading, skill discovery and behavior, and MCP connectivity; update the tested version, validation date, and limitations in `.agents/README.md` |
| Validate a skill | `skills-ref validate .agents/skills/<name>` |

---

## Sources

| Source | Covers | Link |
|--------|--------|------|
| AGENTS.md / AAIF | Shared agent instructions | [agents.md](https://agents.md), [spec repo](https://github.com/agentsmd/agents.md), [AAIF](https://aaif.io/) |
| Agent Skills | SKILL.md format and discovery | [specification](https://agentskills.io/specification), [client implementation guide](https://agentskills.io/client-implementation/adding-skills-support.md) |
| skills-ref | Skill validation CLI | [github.com/agentskills/agentskills](https://github.com/agentskills/agentskills) |
| Vercel Skills CLI | `npx skills add` | [github.com/vercel-labs/skills](https://github.com/vercel-labs/skills) |
| Model Context Protocol | MCP | [modelcontextprotocol.io](https://modelcontextprotocol.io) |
| Claude Code docs | Claude skill discovery | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) |
| Claude Code issue #31005 | Request for native `.agents/` scanning | [github.com/anthropics/claude-code#31005](https://github.com/anthropics/claude-code/issues/31005) |
| Codex docs | Codex skills and project config | [build skills](https://learn.chatgpt.com/docs/build-skills), [config basics](https://learn.chatgpt.com/docs/config-file/config-basic) |
| opencode docs | opencode config, skills, MCP | [opencode.ai/docs](https://opencode.ai/docs/config/) |
| oh-my-pi docs | omp skills, rules, MCP, context files | [github.com/can1357/oh-my-pi/tree/main/docs](https://github.com/can1357/oh-my-pi/tree/main/docs) |
| SSW Rules | Symlinking harness directories to Claude | [ssw.com.au/rules/symlink-agents-to-claude](https://www.ssw.com.au/rules/symlink-agents-to-claude) |
