# config-doctor

A Claude Code plugin that deep-scans and audits a project's Claude configuration — agents, rules, skills, and memory files — then produces a quality report, tooling gap analysis, and memory optimization recommendations.

## What it does

Runs a five-phase audit:

1. **Orientation** — inventories all `.claude/` configuration files
2. **Agent & Rules Quality Audit** — evaluates every agent against a rubric (0–50), flags rule contradictions, duplicates, and gaps
3. **Tooling Gap Analysis** — identifies missing linters, MCP servers, and LSP integrations for the project's tech stack
4. **Memory Optimization** — finds redundant, verbose, and dead rules; projects token savings
5. **Enforcement** — optionally applies safe fixes automatically or all recommendations with confirmation

## Bundled agents

The plugin ships three agents used internally by the audit:

| Agent | Purpose |
|-------|---------|
| `agent-architect` | Evaluates agent and rules quality using a structured rubric |
| `code-quality-scouter` | Audits developer tooling and recommends MCP/LSP additions |
| `memory-optimizer` | Audits memory files for token waste and redundancy |

The `memory-optimizer` is only used if the project does not already have a custom one in `.claude/agents/`. If a project-level `memory-optimizer` exists, it takes priority.

## Usage

### Via slash command

```
/config-doctor
/config-doctor --report-only
/config-doctor --apply-safe
/config-doctor --apply-all
/config-doctor --skip-tooling
/config-doctor --skip-memory
/config-doctor /path/to/other/project
```

### Via natural language (skill auto-trigger)

Ask Claude to run the config doctor, audit your Claude configuration, or check your `.claude/` setup.

## Installation

### From GitHub (recommended)

```bash
/plugin install github:your-username/config-doctor
```

Or pin to a specific release:

```bash
/plugin install github:your-username/config-doctor@v1.0.0
```

### From a local clone

```bash
git clone https://github.com/your-username/config-doctor
cc --plugin-dir ./config-doctor
```

## Arguments reference

| Flag | Effect |
|------|--------|
| `--report-only` | Phases 1–4 only; no changes applied |
| `--apply-safe` | Auto-applies unambiguously safe fixes |
| `--apply-all` | Applies all recommendations (confirms each significant change) |
| `--skip-tooling` | Skips Phase 2 (tooling gap analysis) |
| `--skip-memory` | Skips Phase 3 (memory optimization) |
| `/path/to/project` | Targets a specific directory instead of cwd |
