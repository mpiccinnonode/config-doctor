---
name: audit
description: Deep-scan a project's Claude configuration (.claude/ directory, CLAUDE.md, agents, rules, skills, memory files). Produces a quality report, tooling gap analysis, and memory optimization recommendations — then optionally enforces significant changes.
version: 1.0.0
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash, Agent, "mcp__plugin_config-doctor_serena__*"]
---

You are orchestrating a multiphase Claude configuration audit. Work through the phases below in order. Be explicit about what you are doing at each step so the user can follow along and intervene if needed.

---

## Preflight — Serena Availability Check

Run `uv --version` via Bash. If found, proceed silently to Phase 0.

If not found, prompt the user:
> "Serena MCP is not installed. It reduces token usage by ~60% through semantic code tools. May I install `uv` to enable it?"

- **Approved:** detect OS via `uname -s 2>/dev/null || echo "Windows_NT"`. Run `scripts/install-uv.sh` (Unix/macOS/MINGW) or `scripts/install-uv.ps1` (Windows) via Bash. Verify with `uv --version`, then proceed.
- **Declined or install failed:** proceed with native tools (`Read`, `Glob`, `Grep`) and note Serena's absence in the Phase 4 summary.

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

## Phase 1 — Agent & Configuration Quality Audit (agent-architect)

Use the Agent tool to launch the **agent-architect** subagent with the following prompt:

```text
Audit all Claude configuration files for this project. Check both .claude/agents/ and agents/ directories for agent definitions; check both .claude/skills/ and skills/ for skills; check .claude/rules/ for rules files. Read CLAUDE.md and any memory files under .claude/.

Apply your Configuration Quality Rubric to each agent. Flag agents scoring below 35/50.
Check CLAUDE.md and rules files for duplicate content, contradictions, echo rules, and gaps.

Output a structured report:
## Agent Audit
## Rules Audit
## Skills Audit
## Priority Issues (must fix)
## Recommendations (improvements)

Do NOT apply any changes. Report only.
```

Capture and display the full report from agent-architect before proceeding.

---

## Phase 2 — Tooling Gap Analysis (code-quality-scouter)

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

## Phase 3 — Memory Optimization

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

## Phase 4 — Consolidated Findings

Synthesize all three reports into a single **Config Doctor Summary**:

```text
## Config Doctor Summary

### Agents Run Report
[Report of plugin agents run, with duration and token consumption]

### Critical Issues (must fix now)
[Issues that will cause agent misbehavior, contradictions, or significant token waste]

### High-Priority Improvements
[Impactful changes that are low-risk]

### Tooling Gaps
[Top 3 tooling additions by impact]

### Memory Savings
[Projected token reduction if memory optimizations are applied]

### Risk Assessment
[Changes that require care — could affect agent behavior in ways that need verification]
```

---

## Phase 5 — Enforcement Decision

Present the user with three options:

**A) Report only** — Done. No changes applied. User can act on findings manually.

**B) Apply safe changes automatically** — Apply only changes that are unambiguously correct:

- Remove exact-duplicate content across rule files
- Fix broken markdown formatting in agent files
- Remove obviously dead rules (no activation path exists)
- Add missing sections to agent files that score 0 on a rubric dimension

**C) Apply all recommendations** — Apply all findings including structural changes, agent rewrites, and rule file reorganization. Requires explicit user confirmation before each significant change.

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

- `--report-only` → Force Phase 5 option A regardless of other input
- `--apply-safe` → Force Phase 5 option B
- `--apply-all` → Force Phase 5 option C (will still confirm each significant change)
- `--skip-agents` → Skip Phase 1
- `--skip-tooling` → Skip Phase 2
- `--skip-memory` → Skip Phase 3
- A path argument (e.g. `/path/to/project`) → Use that path as the project root instead of cwd

$ARGUMENTS
