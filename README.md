# ctx-skills

Agent skills that teach coding agents how and when to use [`ctx`](../ctx) — a local-first tool that links product intent (Features, Requirements, Invariants, Decisions) to code with provenance and confidence.

There is one skill (`skills/ctx/`) with a single canonical copy. It is agent-agnostic: the `SKILL.md` format (`name` + `description` frontmatter, Markdown body, triggered by progressive disclosure) is shared across Claude Code, Codex CLI, and Google Antigravity. The `claude/`, `codex/`, and `antigravity/` directories are symlinks into `skills/ctx` so there is exactly one directory to edit.

`skills/ctx/SKILL.md` is the entry point: repository detection, the mandatory per-task pipeline (compile context → check impact → edit → **review before every commit** → **keep `.context/` in sync** → commit → re-index), and a scenario cookbook. `skills/ctx/references/` holds the material that's too deep for the entry point but still needs to travel with the skill into any target repository (since that repository won't have `ctx`'s own `docs/` checked out):

| File | Covers |
| --- | --- |
| `references/commands.md` | Full CLI flag reference for every subcommand, plus `ctx status`'s JSON fields. |
| `references/authoring-context.md` | `.context/` document schema, which document type to pick, canonical-symbol-path rules per language, visibility. |
| `references/onboarding.md` | Bootstrapping a repository onto ctx — by hand, or fully automated by mining Git history/code comments/GitLab and running an AI review pass. |
| `references/federation.md` | Sharing product knowledge and tracing HTTP requests across sibling repositories on the same team. |

## Install

Pick the agent(s) you use. All commands are run from the target project's repository root (the project that uses `ctx`, not this repo).

### Claude Code

Project-scoped (checked into the target repo, shared with the team):

```bash
mkdir -p .claude/skills
cp -r /path/to/ctx-skills/skills/ctx .claude/skills/ctx
```

Personal (available in every project on this machine):

```bash
mkdir -p ~/.claude/skills
cp -r /path/to/ctx-skills/skills/ctx ~/.claude/skills/ctx
```

### Codex CLI

Project-scoped:

```bash
mkdir -p .agents/skills
cp -r /path/to/ctx-skills/skills/ctx .agents/skills/ctx
```

Personal:

```bash
mkdir -p ~/.codex/skills
cp -r /path/to/ctx-skills/skills/ctx ~/.codex/skills/ctx
```

### Google Antigravity

Antigravity currently only reads skills from the user's home directory, not per-project, for both the CLI and the IDE:

```bash
mkdir -p ~/.gemini/skills
cp -r /path/to/ctx-skills/skills/ctx ~/.gemini/skills/ctx
```

## MCP (recommended alongside the skill)

The skill works over either the `ctx` CLI or its MCP server; MCP gives the agent typed tool calls instead of shelling out. Register it once the target repository has run `ctx init` and `ctx index`:

```json
{
  "mcpServers": {
    "ctx": {
      "command": "/absolute/path/to/ctx",
      "args": ["serve", "--mcp"],
      "cwd": "/absolute/path/to/repository"
    }
  }
}
```

- **Claude Code**: `claude mcp add` or add the block above to `.mcp.json` in the target repo.
- **Codex CLI**: `codex mcp add ctx -- /absolute/path/to/ctx serve --mcp` (or the equivalent `[mcp_servers.ctx]` table in `~/.codex/config.toml`), with `cwd` set via the command's working directory.
- **Antigravity**: add the same shape to `mcp_config.json` (`~/.gemini/config/mcp_config.json` globally, or `.agents/mcp_config.json` per workspace), or install through the built-in MCP Store if `ctx` is published there.

Exact MCP config keys vary by agent version; check the agent's own docs if a field name above has changed. The skill itself does not depend on MCP being configured — it works over the CLI too, MCP is strictly an ergonomics upgrade.

## Why one skill instead of three

`ctx`'s value to an agent is the same regardless of which harness is driving it: compile context before editing, check blast radius before touching a symbol, review the diff before finishing. Splitting that into three near-duplicate files per agent would just be a maintenance liability. If a given agent ever needs materially different instructions (not just a different install path), fork the content into an agent-specific file at that point — not before.
