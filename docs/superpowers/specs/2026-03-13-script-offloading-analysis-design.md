# Script-Offloading Analysis in Audit Phase 2

**Date:** 2026-03-13
**Status:** Approved
**Branch:** 1-implement-anthropic-skills-testing

## Problem

Skills often spend significant tokens on work that doesn't require LLM reasoning — formatting reports, collecting file metadata, validating YAML frontmatter, checking version sync. Pre-written scripts colocated in the skill's folder could handle this deterministic work at near-zero token cost while maintaining or improving output quality. Currently, no part of the audit identifies these opportunities.

## Design

### Overview

The audit's Phase 2 (Skills Quality Audit) gains a second parallel agent dispatch. The **code-quality-scouter** runs alongside the existing **skill-evaluator**, scoped specifically to identify script-offloading opportunities in skill files.

- The skill-evaluator's rubric (0-50, five dimensions) is unchanged.
- The scouter's findings appear as a separate section in the Phase 5 summary.
- No scoring interaction between the two agents — they run independently.

### Script-offloading categories

The scouter evaluates each skill for three categories of offloadable work:

1. **Output formatting** — Report templates, structured output generation, markdown table assembly. A script could accept raw findings as JSON and produce the final formatted report.
2. **Data collection/validation** — Multi-command sequences that gather file counts, sizes, checksums, directory listings, or config values. A single script could return structured JSON instead of the LLM running N Bash commands and parsing output.
3. **Deterministic logic** — YAML frontmatter validation, version sync checks, regex-based pattern matching, duplicate detection, or any rule that can be expressed as a boolean check without LLM reasoning.

### Phase 2 changes

Currently Phase 2 dispatches only the skill-evaluator. After this change, it dispatches both agents in parallel using two Agent tool calls in a single message:

1. **skill-evaluator** — existing prompt, unchanged
2. **code-quality-scouter** — new scoped prompt (see below)

Both agents' reports are captured and displayed before proceeding to Phase 3.

### Scouter prompt (dispatched from audit Phase 2)

```text
Analyze all skill files in this project for script-offloading opportunities.
Check both .claude/skills/ and skills/ directories (recursively) for skill
definitions (SKILL.md files).

For each skill, identify work currently done by the LLM that could instead be
handled by a pre-written script colocated in the skill's folder. Evaluate
three categories:

1. **Output formatting** — report templates, structured output, markdown
   assembly that follows a fixed pattern
2. **Data collection/validation** — multi-command sequences gathering file
   metadata, config values, or directory structure that could be a single
   script returning JSON
3. **Deterministic logic** — YAML validation, version checks, regex matching,
   duplicate detection, or any rule expressible as a boolean check without
   LLM reasoning

Output a structured report:

## Script Offloading Analysis

### <skill-name>

| Opportunity | Type | Token Savings | Implementation Effort |
|---|---|---|---|
| <description> | <Output formatting / Data collection / Deterministic logic> | <estimated tokens saved per run> | <Low / Medium / High> |

#### Details
[Brief description of each opportunity: what the script would do, what
inputs it needs, what it returns]

### Summary
[Total estimated token savings across all skills, top 3 highest-impact
opportunities]

If no offloading opportunities are found for a skill, state that explicitly.
Do NOT apply any changes. Report only.
```

### Phase 5 changes

The Consolidated Findings template gains a new section after "Memory Savings":

```text
### Script Offloading Opportunities
[Top opportunities from the scouter's analysis, ordered by estimated token
savings. For each: skill name, opportunity description, category, estimated
savings, effort level.]
```

### Phase 6 changes

Under option C (Apply all recommendations), script-offloading recommendations are presented to the user for acknowledgment but **not auto-implemented**. Writing scripts is outside the audit's enforcement scope — the report serves as an actionable backlog.

No changes to options A or B.

### `--skip-skills` and `--phase` behavior

The scouter dispatch is tied to the `skills` phase name. If `--skip-skills` is active or `--phase` does not include `skills`, both the skill-evaluator and the scouter are skipped together. No new flags are introduced.

### Files modified

| File | Change |
|---|---|
| `skills/audit/SKILL.md` | Phase 2: add parallel scouter dispatch. Phase 5: add Script Offloading Opportunities section. Phase 6 option C: add note about script recommendations. |
| `CLAUDE.md` | Update audit skill description to mention dual Phase 2 dispatch. |

### Files NOT modified

| File | Reason |
|---|---|
| `agents/code-quality-scouter.md` | The scoped prompt comes from the audit skill, not the agent definition. The scouter's general-purpose description and capabilities remain unchanged. |
| `agents/skill-evaluator.md` | Rubric unchanged. No scoring interaction with the scouter. |

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Scouter token estimates are speculative | The table labels them as estimates; users understand these are approximate. The value is in identifying opportunities, not precise numbers. |
| Scouter may not find opportunities in simple skills | The prompt explicitly instructs it to state "no opportunities found" per skill, so the report is still useful. |
| Parallel dispatch increases Phase 2 wall time if scouter is slow | Both agents run on sonnet; the scouter's task is lighter than the skill-evaluator's, so it should finish first or around the same time. |

## Out of scope

- Auto-generating scripts from the recommendations (future work)
- Adding a new rubric dimension to the skill-evaluator for script offloading
- Changes to the code-quality-scouter's agent definition
- New CLI flags for the audit skill
