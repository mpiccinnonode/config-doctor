---
name: audit
description: Deep-scan a project's Claude configuration (.claude/ directory, CLAUDE.md, agents, rules, skills, memory files). Produces a quality report, tooling gap analysis, and memory optimization recommendations — then optionally enforces significant changes.
version: 1.0.0
---

You are orchestrating a multi-phase Claude configuration audit. Work through the phases below in order. Be explicit about what you are doing at each step so the user can follow along and intervene if needed.

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

```
You are performing a full Claude configuration audit for this project.

Your tasks:
1. Read CLAUDE.md and all files under .claude/rules/ (recursively).
2. Read all agent definitions under .claude/agents/ (project-level).
3. Read all skill definitions under .claude/skills/ (project-level, if any).
4. Read any memory files under .claude/ (MEMORY.md or memory/ subdirectory).

For each agent definition, apply your full Configuration Quality Rubric (Identity & Persona, Instructions Clarity, Operational Completeness, Purposefulness, Project Alignment — 0–10 each). Flag agents scoring below 35/50 for mandatory improvement.

For the rules structure:
- Check CLAUDE.md is a lean index, not a content dump
- Verify guardrails.md contains only unique do's and don'ts not echoed elsewhere
- Identify any rules that contradict each other or overlap significantly
- Identify rules with no plausible activation scenario (candidates for removal)
- Flag any rules missing from the structure given the project's tech stack

For skills (if present):
- Verify each skill has a clear triggering condition
- Check that skill prompts are self-contained and don't assume ambient context

Output a structured report with:
## Agent Audit
[Per-agent rubric scores + issues]

## Rules Audit
[Findings per file — duplicates, contradictions, gaps, verbosity]

## Skills Audit
[Findings per skill]

## Priority Issues (must fix)
[Ordered list — most critical first]

## Recommendations (improvements)
[Ordered list — highest impact first]

Do NOT apply any changes. Report only.
```

Capture and display the full report from agent-architect before proceeding.

---

## Phase 2 — Tooling Gap Analysis (code-quality-scouter)

Use the Agent tool to launch the **code-quality-scouter** subagent with the following prompt:

```
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
```
Perform a full memory audit of this project's Claude memory files:
- CLAUDE.md
- All files under .claude/rules/ (recursively)
- Any MEMORY.md or memory/ files under .claude/

Apply your full Analysis Process (Steps 1–4: inventory, codebase scan, redundancy detection, verbosity analysis).

Output your full Memory Audit Report including:
- Token Footprint Summary table
- Priority Findings (highest token savings first)
- Secondary Optimizations
- Dormant Rules Review
- Non-Negotiable sections
- Optimization Summary with projected token savings %

Do NOT edit any files. Report only.
```

Capture and display the memory audit before proceeding.

---

## Phase 4 — Consolidated Findings

Synthesize all three reports into a single **Config Doctor Summary**:

```
## Config Doctor Summary

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
- `--skip-tooling` → Skip Phase 2
- `--skip-memory` → Skip Phase 3
- A path argument (e.g. `/path/to/project`) → Use that path as the project root instead of cwd

$ARGUMENTS
