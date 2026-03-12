# CLAUDE.md

config-doctor is a Claude Code plugin. No build step or test suite — content is markdown files executed by Claude Code, with small helper scripts for platform-specific installation tasks.

## Repository structure

```text
.claude-plugin/
  plugin.json        # Plugin metadata (name, version, author, repo)
  marketplace.json   # Marketplace registry entry for distribution

agents/              # Bundled subagents — deployed to the user's .claude/agents/ on install
  agent-architect.md      # Evaluates agent/rules quality via a 0–50 rubric (runs on claude opus)
  code-quality-scouter.md # Audits developer tooling and recommends MCP/LSP additions (sonnet)
  memory-optimizer.md     # Audits memory files for token waste and redundancy (sonnet)
  skill-evaluator.md      # Evaluates skill files for quality via a 0-50 rubric (sonnet)

skills/audit/
  SKILL.md           # The full multi-phase audit orchestration logic (7 phases)

scripts/
  install-uv.sh      # Installs uv on Unix/macOS (run by the audit skill preflight)
  install-uv.ps1     # Installs uv on Windows (run by the audit skill preflight)

.mcp.json            # Serena MCP server declaration (uvx launcher)
.gitignore           # OS and IDE artifact exclusions
.markdownlint.yaml   # Markdown lint rules for CI validation
```

## MCP dependency: Serena

All four subagents are instructed to prefer **Serena MCP tools** (`mcp__plugin_config-doctor_serena__*`) over native file tools for efficiency. Serena is declared in `.mcp.json` and launched via `uvx` from `git+https://github.com/oraios/serena`. The `uv` package manager is a prerequisite.

Agents fall back to `Read`, `Glob`, and `Grep` if Serena is unavailable.

## Key conventions when editing this plugin

- **Agent frontmatter**: each agent file requires `name:`, `description:` (with `<example>` blocks), and `model:` fields. `agent-architect` runs on `opus`; others run on `sonnet`.
- **Skill frontmatter**: `skills/audit/SKILL.md` requires `name:`, `description:`, `version:`, and `allowed-tools:` (currently includes `mcp__plugin_config-doctor_serena__*` wildcard). The `argument-hint:` field controls slash command autocomplete.
- **`$ARGUMENTS`**: the skill receives user flags (`--report-only`, `--apply-safe`, `--apply-all`, `--skip-agents`, `--skip-skills`, `--skip-tooling`, `--skip-memory`, or a path) via the `$ARGUMENTS` placeholder at the end of `SKILL.md`.
- **Version sync**: `plugin.json` and `SKILL.md` both declare a `version` field. Keep them in sync — `plugin.json` is the source of truth.
- **Scripts**: only small, focused shell scripts in `scripts/` are permitted — for platform-specific install tasks that cannot be expressed in markdown. Do not introduce package managers, build tools, or compiled assets.
