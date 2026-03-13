---
name: audit
version: "1.0.1"
description: Deep-scan a project's Claude configuration (.claude/ directory, CLAUDE.md, agents, rules, skills, memory files). Produces a quality report, tooling gap analysis, and memory optimization recommendations — then optionally enforces significant changes.
argument-hint: "[--report-only | --apply-safe | --apply-all] [--phase=agents,skills,...] [--skip-agents] [--skip-skills] [--skip-tooling] [--skip-memory] [/path/to/project]"
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash, Agent]
---

You are orchestrating a multiphase Claude configuration audit. Work through the phases below in order. Be explicit about what you are doing at each step so the user can follow along and intervene if needed.

---

## Preflight — Serena Availability Suggestion

Check whether Serena MCP tools are available in this session by looking for tools matching `mcp__serena__*` or `mcp__plugin_*_serena__*`.

- **If found:** note "Serena MCP detected — agents will use structured semantic tools for lower token cost" and proceed to Phase 0.
- **If not found:** proceed with native tools (`Read`, `Glob`, `Grep`). In the Phase 5 summary, include a suggestion: "Consider adding Serena MCP (https://github.com/oraios/serena) for ~60% token savings through semantic code tools."

---

## Phase 0 — Orientation

Determine the project root by locating `CLAUDE.md` and the `.claude/` directory relative to the current working directory. List their contents so you know what exists:

- `.claude/rules/` — all rule files (recursive)
- `.claude/agents/` — project-level agent definitions
- `.claude/skills/` — project-level skill definitions (if any)
- `.claude/projects/*/memory/` or any `MEMORY.md` under `.claude/`
- `CLAUDE.md` at the project root

Report a brief inventory table (file path, rough size in lines) before proceeding.

---

## Phase 1 — Agent & Rules Quality Audit (agent-architect)

Use the Agent tool to launch the **agent-architect** subagent with the following prompt:

```text
Audit all Claude configuration files for this project. Check both .claude/agents/ and agents/ directories for agent definitions; check .claude/rules/ for rules files. Skills are evaluated separately by the skill-evaluator agent. Read CLAUDE.md and any memory files under .claude/.

Apply your Configuration Quality Rubric to each agent. Flag agents scoring below 35/50.
Check CLAUDE.md and rules files for duplicate content, contradictions, echo rules, and gaps.

Output a structured report:
## Agent Audit
## Rules Audit
## Priority Issues (must fix)
## Recommendations (improvements)

Do NOT apply any changes. Report only.
```

Capture and display the full report from agent-architect before proceeding.

---

## Phase 2 — Skills Quality Audit (skill-evaluator)

Use the Agent tool to launch the **skill-evaluator** subagent with the following prompt:

```text
Audit all skill files for this project. Check both .claude/skills/ and skills/
directories (recursively) for skill definitions.

Apply your Skills Quality Rubric to each skill. Flag skills scoring below 35/50.

Output a structured report:
## Skills Inventory
## Per-Skill Evaluation (table: Dimension / Score / Notes)
## Priority Issues (must fix)
## Recommendations (improvements)

Do NOT apply any changes. Report only.
```

Capture and display the full skills report before proceeding.

---

## Phase 3 — Tooling Gap Analysis (code-quality-scouter)

Use the Agent tool to launch the **code-quality-scouter** subagent with the following prompt:

```text
Audit this project's developer tooling and Claude Code integrations.

Focus on:
1. What code quality tools are configured (linters, formatters, type checkers, dead-code finders, bundle analyzers)?
2. What MCP servers are active (check .claude/settings.json or mcp.json if present)?
3. What pre-commit hooks exist?
4. What gaps exist given the project's tech stack?
5. Are there MCP servers or LSP tools that would meaningfully improve Claude Code's performance in this codebase?

Output format:
## Current Toolchain
[What's configured]

## Active MCP/LSP Integrations
[What's live in Claude Code]

## High-Priority Gaps
[Tools missing, ordered by impact]

## Quick Wins
[Tools that can be added in under 30 minutes with high payoff]

Do NOT install or configure anything. Report only.
```

Capture and display the full tooling report before proceeding.

---

## Phase 4 — Memory Optimization

Check whether a `memory-optimizer` agent exists in the **project's** `.claude/agents/` directory.

**If a project-level memory-optimizer is found:** use it — it may be customized for this project's conventions. Launch it via the Agent tool with the prompt below.

**If no project-level memory-optimizer is found:** use the **bundled `memory-optimizer`** agent that ships with this plugin. Launch it via the Agent tool with the same prompt below.

Prompt to use in either case:

```text
Audit the memory files for this project: CLAUDE.md, all files under .claude/rules/ (recursively), and any MEMORY.md or memory/ files under .claude/.

Output your full Memory Audit Report. Do NOT edit any files.
```

Capture and display the memory audit before proceeding.

---

## Phase 5 — Consolidated Findings

Synthesize all four reports into a single **Config Doctor Summary**:

```text
## Config Doctor Summary

### Agents Run Report
[Report of plugin agents run, with duration and token consumption]

### Critical Issues (must fix now)
[Issues that will cause agent misbehavior, contradictions, or significant token waste. Include skills scoring below 25/50.]

### High-Priority Improvements
[Impactful changes that are low-risk. Include skills scoring 25-34/50.]

### Tooling Gaps
[Top 3 tooling additions by impact]

### Memory Savings
[Projected token reduction if memory optimizations are applied. Include token savings from skill prompt compression.]

### Risk Assessment
[Changes that require care — could affect agent behavior in ways that need verification]
```

---

## Phase 6 — Enforcement Decision

Present the user with three options:

**A) Report only** — Done. No changes applied. User can act on findings manually.

**B) Apply safe changes automatically** — Apply only changes that are unambiguously correct:

- Remove exact-duplicate content across rule files
- Fix broken markdown formatting in agent files
- Remove obviously dead rules (no activation path exists)
- Add missing sections to agent files that score 0 on a rubric dimension
- Fix frontmatter issues in skill files (missing fields, description format, naming violations)

**C) Apply all recommendations** — Apply all findings including structural changes, agent rewrites, and rule file reorganization. Requires explicit user confirmation before each significant change.

- Rewrite skills scoring below 35/50 using skill-evaluator's suggested revisions

Ask the user which option they want before proceeding. Default to A if no response.

---

## Enforcement Guidelines (for options B and C)

When applying changes:

1. **Always read the file before editing** — never write blind
2. **One change at a time** — describe what you are about to change and why, then apply it
3. **Preserve intent** — when rewriting for brevity, verify the rule's meaning is unchanged
4. **Agent rewrites** — when rewriting an agent that scored < 35/50, use agent-architect to produce the improved version, then apply it
5. **Never delete a rule file entirely** without explicit user confirmation
6. **After each batch of changes**, summarize what was changed and what remains

---

## User Arguments

If arguments were passed when invoking this skill (`$ARGUMENTS`), interpret them as follows:

### Phase selection (`--phase`)

- `--phase=<name>` → Run **only** the listed phase(s). Accepts a comma-separated list of phase names.
- Valid phase names: `agents`, `skills`, `tooling`, `memory`
- Examples: `--phase=skills`, `--phase=agents,skills`, `--phase=tooling,memory`
- Preflight and Phase 0 (Orientation) **always run** regardless of phase selection — they are infrastructure.
- Phase 5 (Consolidated Findings) and Phase 6 (Enforcement Decision) **always run** after the selected phases, adapting their output to cover only the phases that were executed.

**Phase name mapping:**

| Phase name | Phase |
|---|---|
| `agents` | Phase 1 — Agent & Rules Quality Audit |
| `skills` | Phase 2 — Skills Quality Audit |
| `tooling` | Phase 3 — Tooling Gap Analysis |
| `memory` | Phase 4 — Memory Optimization |

### Skip flags

- `--skip-agents` → Skip Phase 1
- `--skip-skills` → Skip Phase 2
- `--skip-tooling` → Skip Phase 3
- `--skip-memory` → Skip Phase 4

### Conflict resolution

If both `--phase` and `--skip-*` flags are present, `--phase` takes priority. The `--skip-*` flags are ignored and a note is shown to the user explaining that `--phase` was used instead.

### Enforcement mode

- `--report-only` → Force Phase 6 option A regardless of other input
- `--apply-safe` → Force Phase 6 option B
- `--apply-all` → Force Phase 6 option C (will still confirm each significant change)

### Other

- A path argument (e.g. `/path/to/project`) → Use that path as the project root instead of cwd

$ARGUMENTS
