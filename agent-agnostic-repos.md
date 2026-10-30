# Agent-Agnostic Repository Guide

How to make a repository work with any AI coding agent without vendor lock-in: setting up a new repo (Part A) and migrating an existing one (Part B).

---

## The Core Principle

Every piece of agent configuration falls into one of three categories:

| Category | Strategy | Examples |
|----------|----------|----------|
| **Portable** (same format across agents) | Store once at the root or in `.agents/`. Agents that don't read it natively reach it through a symlink (skills) or an import (`CLAUDE.md` → `@AGENTS.md`) | `AGENTS.md`, `SKILL.md` |
| **Generated** (same data, different format) | Canonical source in `.agents/`; a script renders each agent's file | MCP config |
| **Agent-specific** (no cross-agent equivalent) | Leave in the agent's own directory; keep it thin and refer to portable sources instead of copying them | `.omp/rules/*.md`, `.claude/settings.json` |

---

## Part A: New Repo Template

### A.1 Directory Layout

```
repo/
  AGENTS.md                  # Shared instructions
  CLAUDE.md                  # "@AGENTS.md" + Claude-only overrides
  .agents/
    README.md                # How to maintain this directory (not AGENTS.md, see A.8)
    skills/<name>/SKILL.md   # Shared skills, with optional references/ and scripts/
    skills-local/.gitkeep    # Private skills (gitignored)
    mcp/servers.json         # Canonical MCP servers
    scripts/
      sync-skills.sh         # Links skills into agent dirs
      sync-mcp.sh            # Renders agent MCP configs
  .claude/
    skills/<name> -> ../../.agents/skills/<name>
    settings.json            # Agent-specific
  .omp/                      # oh-my-pi
    rules/                   # Agent-specific
    mcp.json                 # Generated
  .opencode/agents/          # Agent-specific
  opencode.json              # Agent-specific; `mcp` key generated
  .codex/config.toml         # Agent-specific; `[mcp_servers.*]` tables generated
  .mcp.json                  # Generated (Claude Code)
  .gemini/settings.json      # Optional: makes Gemini CLI read AGENTS.md
```

### A.2 Instructions (AGENTS.md)

[AGENTS.md](https://agents.md), the AAIF/Linux Foundation standard, holds the project overview, layout, setup commands, coding standards, git conventions, and a pointer to `.agents/README.md`. Codex, oh-my-pi (omp), Copilot, Devin and many other agents read it natively. Two need an adapter:

- **Claude Code** reads `CLAUDE.md`. Make it an import, followed by any Claude-only overrides:

  ```markdown
  @AGENTS.md
  ```

- **Gemini CLI** reads `GEMINI.md` unless `.gemini/settings.json` says otherwise: `{"context": {"fileName": ["AGENTS.md", "GEMINI.md"]}}`.

For directory-scoped context in nested packages, add files like `src/package/AGENTS.md`; Codex and omp load every `AGENTS.md` from the repo root down to the working directory. Claude Code ignores `AGENTS.md` once a `CLAUDE.md` exists, so put a `CLAUDE.md` containing `@AGENTS.md` next to each nested one.

### A.3 Skills (SKILL.md)

Skills are reusable workflows agents can invoke (`/deploy`, `/learn`). The [Agent Skills spec](https://agentskills.io/specification) format is portable:

```yaml
---
name: my-skill          # Lowercase and hyphens, matches the directory name
description: What this skill does and when to use it.
---

Step-by-step instructions for the agent...
```

Keep `SKILL.md` under 500 lines and move details to `references/`.

The format is portable; behavior isn't always. Agent-specific features (Claude's `context: fork`, inline shell-command substitution, `allowed-tools`, whose support varies even per the spec) are ignored or interpreted differently by other agents. Shared skills should contain only portable instructions and resources. A skill that depends on one agent's features stays in that agent's skills directory.

**Discovery.** Only agents that don't scan `.agents/skills/` get links. Don't link into the others: the extra copy is redundant, and Codex lists it as a duplicate skill.

| Agent | Reads `.agents/skills/`? | Sync target |
|-------|--------------------------|-------------|
| Claude Code | No | `.claude/skills/` |
| Codex | Yes (cwd up to repo root) | None |
| omp | Yes (cwd up to repo root) | None |
| opencode | Yes (cwd up to git worktree) | None |

omp and opencode also scan `.claude/skills/` and should treat the links as the same skill; check this with your versions.

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
# Agents that don't discover .agents/skills/ natively
TARGETS=(".claude/skills")
# Agents that do (omp, opencode) and leftovers from older setups: never sync here, only
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

# Remove old links from agents that read .agents/skills/ natively (they duplicate skills)
for native_dir in "${NATIVE[@]}"; do
    if [ -L "$REPO_ROOT/$native_dir" ]; then
        # Never prune through a dir alias: it could point at .claude/skills or .agents/skills
        echo "ERROR: $native_dir is a symlink; this agent reads .agents/skills/ natively, so remove it." >&2; exit 2
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

- It runs on the bash 3.2 that ships with macOS.
- It only touches links of the exact form it creates (`../../.agents/skills/<name>`); anything else in the way stops it with exit 2. List hand-made, machine-local links (e.g., to `~/.agents/skills/`) in `.git/info/exclude` so the staging check ignores them.
- It handles the `.agents/skills/` next to it. Skills for one nested package belong in that package's own `.agents/skills/`, with its own copy of the script and hook entry; moving them to the root makes them visible repo-wide.
- It only makes symlinks. On Windows, run it from Git Bash or WSL and enable `core.symlinks` and Developer Mode (or admin rights) before cloning; otherwise links check out as text files and the script stops with exit 2. Agents or machines that can't use symlinks need a copy mode, which must track the copies it owns and remove stale ones.

**Third-party skills** ([Vercel skills CLI](https://github.com/vercel-labs/skills)):

```bash
npx skills add <repo>@<skill-name> -y
.agents/scripts/sync-skills.sh
```

Which agent dirs the CLI links into depends on what it detects, so always run the sync script afterwards. If the CLI wrote a copy into `.claude/skills/`, the script stops: check that the copy matches (`diff -r .claude/skills/<name> .agents/skills/<name>`), delete it, and rerun.

**Private skills** go in `.agents/skills-local/` (gitignored). If you extend the script to link them, don't commit those links.

### A.4 MCP Configuration

MCP server definitions hold the same data for every agent in a different format, so they are generated from `.agents/mcp/servers.json`:

| Agent | File | Format |
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

A bare map of server names, with no `mcpServers` wrapper. Each server is `stdio` (`command`, `args`, `env`) or `http` (`url`, `headers`). Never put secrets in it. Use placeholders instead, in one of two forms: a whole value `${NAME}` (in `env` or a header), or `Bearer ${NAME}` in an `Authorization` header.

**The generator** (`sync-mcp.sh`) is not included here; a short Python or `jq` script is typical. It should:

- render each server for each agent based on its `type`:
  - Claude Code: canonical fields under `mcpServers`; placeholders work as-is
  - opencode: `local`/`remote`; `command` and `args` merged into one `command` array; `environment` instead of `env`; `{env:NAME}` instead of `${NAME}`
  - Codex: an `env` entry `${NAME}` with the same name becomes `env_vars = ["NAME"]`; `Bearer ${NAME}` becomes `bearer_token_env_var = "NAME"`; other placeholder headers go in `env_http_headers`, static ones in `http_headers`
  - omp: check its docs for placeholder support
- never expand placeholders into values
- in `opencode.json` and `.codex/config.toml`, rewrite only the `mcp` key or the `[mcp_servers.*]` tables (with a JSONC-aware tool for `opencode.jsonc`), and remove only servers it generated
- write files atomically
- support `check`: render to a temporary location, compare, exit non-zero on differences, change nothing
- fail on a field or placeholder it can't map, instead of dropping it

For servers the canonical format doesn't cover (SSE, OAuth, agent-specific options), either extend the format and the generator, or configure them by hand per agent; in that case the generator must track which server names it owns.

**Codex trust:** Codex loads `.codex/` only for trusted projects, so review the checkout and trust it. Writing the servers into `~/.codex/config.toml` instead makes them apply to every project and can collide with existing names. If you offer that, make it an explicit opt-in (e.g., an `install-codex` command writing a block between `# BEGIN <repo> managed MCP` / `# END <repo> managed MCP` comments) that refuses on name collisions, with a matching uninstall.

### A.5 Agent-Specific Files (Leave As-Is)

| File | Agent | Contents |
|------|-------|----------|
| `.omp/rules/*.md` | omp | Rules triggered by `description`, `alwaysApply`, `condition`, `globs`. Loaded only from the directory omp starts in, so launch omp from the repo root |
| `opencode.json` (except `mcp`), `.opencode/agents/*.md` | opencode | Settings, subagents |
| `.codex/config.toml` (except `mcp_servers`) | Codex | Models, sandbox, approvals |
| `.claude/settings.json`, `.claude/agents/*.md` | Claude Code | Hooks, subagents |
| `.claude/settings.local.json` | Claude Code | Machine-local permissions (gitignored) |

Don't sync between these formats. Put knowledge every agent needs in `AGENTS.md`, and keep agent-specific files thin: an `.omp/rules/` rule adds only its trigger and links to the doc with the guidance, and a hook in `.claude/settings.json` calls a script in `.agents/scripts/`.

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
        always_run: true  # Also catches hand edits to generated files and generator changes
```

When `sync-skills.sh` changes something, or a link or new skill is untracked or unstaged, the commit stops; stage and commit again. An untracked directory in `.agents/skills/` blocks every commit, so keep drafts in `.agents/skills-local/`.

The check covers what is being committed. During `git commit`, pre-commit sets unstaged edits to tracked files aside, so the hook sees the staged links; `pre-commit run --all-files` doesn't, so it also reports unstaged edits.

Hooks only run after `pre-commit install` and are skipped by `--no-verify`, so also run `pre-commit run --all-files` in CI.

### A.7 .gitignore

```gitignore
.agents/skills-local/*
!.agents/skills-local/.gitkeep
.claude/settings.local.json
.claude/worktrees/
```

In a **whitelist repo** (`*` ignores everything), put the allow-list first and the machine-local ignores after it. Git uses the last matching pattern, so `!.agents/**` would otherwise re-include private files:

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

This lets any agent maintain the directory. Name it `README.md`, not `AGENTS.md`: omp loads `.agents/AGENTS.md` at the same level as the root `AGENTS.md`, which it then hides. Link to it from the root `AGENTS.md`.

```markdown
# .agents/

Agent-agnostic configuration following the [Agent Skills](https://agentskills.io)
and [AGENTS.md](https://agents.md) conventions.

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

- Edit shared skills here, never through their links in `.claude/skills/`.
  Claude-only skills stay as real directories in `.claude/skills/`
- Edit MCP servers only in `mcp/servers.json`; the agent MCP configs are generated
- The pre-commit hook re-syncs skills and stops the commit while a link or new skill is missing from it
```

### A.9 New Repo Checklist

1. Create `AGENTS.md`, `CLAUDE.md` (`@AGENTS.md`), and `.gemini/settings.json` if you use Gemini CLI
2. Create `.agents/` as in A.1, including `README.md` (A.8)
3. Add `sync-skills.sh`, write `sync-mcp.sh` (A.4), and `chmod +x .agents/scripts/*.sh`
4. Add skills and run `sync-skills.sh`. It exits 1 after creating links and until they are staged, so don't chain it with `&&`
5. Run `sync-mcp.sh`
6. Update `.gitignore` (A.7)
7. Add `.pre-commit-config.yaml`, install pre-commit (e.g., `uv tool install pre-commit`), and run `pre-commit install`; every clone needs this
8. Stage everything, run `pre-commit run --all-files`, and check that each agent lists the skills and MCP servers

---

## Part B: Migration Guide

### B.1 Assessment

```bash
# Agent config files, at any depth
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
| `CLAUDE.md` with full instructions | Portable | Move content to `AGENTS.md`, leave an `@AGENTS.md` import |
| Real skill dirs in `.claude/`, `.omp/`, `.opencode/`, `.codex/` `skills/` | Portable unless they use agent-specific features (A.3) | Move to `.agents/skills/`, or leave in place if agent-specific |
| `.agents/skills/*/` | Portable | None |
| `.mcp.json`, `.omp/mcp.json`, `opencode.json` `mcp` key, `.codex/config.toml` `[mcp_servers.*]` | Generated | Merge into `.agents/mcp/servers.json`, then generate |
| `.omp/rules/`, `.claude/settings*.json`, other `opencode.json` and `.codex/config.toml` settings, `.opencode/agents/` | Agent-specific | Leave |

### B.3 Migration Steps by Layer

Go from lowest to highest risk:

1. **Instructions.** Keep an existing `AGENTS.md` as canonical. If only `CLAUDE.md` exists, rename it to `AGENTS.md` and recreate `CLAUDE.md` as an import. If both exist, merge into `AGENTS.md` and reduce `CLAUDE.md` to the import plus overrides.
2. **Skills.**
   - Check every skills directory from B.2, including in nested packages.
   - Leave skills that rely on agent-specific features where they are.
   - Merge same-named skills with different content by hand.
   - `git mv` each portable skill's whole directory (it can hold `references/` and `scripts/`) to the `.agents/skills/` at the same level: the root's, or the nested package's (A.3).
   - Run `sync-skills.sh` and add the hook.
3. **MCP.**
   - Collect servers from every agent's config; one file rarely has them all.
   - Convert them to the canonical format (A.4): take the entries without the wrapper (e.g., `jq .mcpServers .mcp.json`), translate agent-specific fields back (opencode `local` with a command array becomes `stdio` with `command` and `args`), and replace literal secrets with placeholders.
   - Resolve same-named servers that differ by hand.
   - Write `sync-mcp.sh`, run it, and diff the outputs against the originals; every difference should be intentional.
   - Add the check to pre-commit and CI.
4. **Cleanup.** Add `.agents/README.md` (renaming an existing `.agents/AGENTS.md`, which hides the root `AGENTS.md` in omp), update `.gitignore`, and run `pre-commit run --all-files`.

### B.4 Archetype-Specific Guidance

**Partially migrated code project** (has `.agents/skills/`, a root `AGENTS.md`, maybe a script that copies skills around): replace the copy script with `sync-skills.sh` (symlinks only, see A.3), add the hooks, make MCP generation cover every agent, and land it all in one commit.

**Repo with submodules and agent-specific rules:**

- Leave `.omp/rules/` as is (A.5); the parent's rules handle only meta-level navigation.
- Migrate each submodule on its own instead of centralizing its config in the parent.
- Agents find project MCP config relative to where they start, not where the edited files live, and each searches differently: omp reads only its start directory, opencode searches up to the Git boundary. A session started at the parent root uses the parent's servers even while editing a submodule; one started at a submodule's root uses the submodule's (for omp, only at that exact root). User-level configs add servers in both cases. Give a submodule its own `.agents/mcp/servers.json` if people start agents there.

**Coordination hub** (not a code project; many sub-projects in their own repos):

- In a whitelist `.gitignore` (A.7), allow every agent directory, and add new ones as you adopt new agents.
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
Do multiple agents read this file identically?
  YES -> Portable: keep one copy in .agents/ or at the root; link or import it where needed.
  NO  -> Same data in a different format?
    YES -> Generated: canonical source in .agents/, rendered by a script.
    NO  -> Agent-specific: leave it, referring to portable sources where possible.
```

### B.6 Ongoing Maintenance

| Task | How |
|------|-----|
| Add or install a skill | Create it in `.agents/skills/` or `npx skills add`, run `sync-skills.sh`, commit skill and links |
| Add an MCP server | Edit `.agents/mcp/servers.json`, run `sync-mcp.sh`, commit generated files |
| Support a new agent | Add its rendering to `sync-mcp.sh`; add it to `TARGETS` in `sync-skills.sh` unless it reads `.agents/skills/` natively |
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
| SSW Rules | Symlinking agent dirs to Claude | [ssw.com.au/rules/symlink-agents-to-claude](https://www.ssw.com.au/rules/symlink-agents-to-claude) |
