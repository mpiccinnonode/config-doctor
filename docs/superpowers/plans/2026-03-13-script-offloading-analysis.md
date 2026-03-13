# Script-Offloading Analysis Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a parallel code-quality-scouter dispatch to the audit's Phase 2, identifying script-offloading opportunities in skill files.

**Architecture:** The audit skill's Phase 2 section gains a second Agent tool call (scouter) alongside the existing skill-evaluator dispatch. Phase 5 and 6 templates are updated to surface the scouter's findings. No agent definition files change.

**Tech Stack:** Markdown (Claude Code plugin — no build step, no tests)

**Spec:** `docs/superpowers/specs/2026-03-13-script-offloading-analysis-design.md`

---

## Chunk 1: SKILL.md Phase 2 rewrite

### Task 1: Rewrite Phase 2 heading and body for dual dispatch

**Files:**

- Modify: `skills/audit/SKILL.md` (Phase 2 section, currently lines 62–81)

All edits use content-based find/replace — line numbers are for reference only.

- [ ] **Step 1: Update the Phase 2 heading**

Find:

```text
## Phase 2 — Skills Quality Audit (skill-evaluator)
```

Replace with:

```text
## Phase 2 — Skills Quality & Efficiency Audit (skill-evaluator + code-quality-scouter)
```

- [ ] **Step 2: Replace the Phase 2 body**

Find the entire block from `Use the Agent tool to launch the **skill-evaluator**` through `Capture and display the full skills report before proceeding.` (lines 64–81 in the original file). Replace it with the following prose and fenced code blocks — note: the replacement is raw markdown prose with two inner fenced code blocks, matching the style of Phases 1, 3, and 4:

<!-- BEGIN REPLACEMENT TEXT -->

Use the Agent tool to launch **both** of the following subagents **in parallel** (two Agent tool calls in a single message):

**skill-evaluator:** (write as ` ```text ` in SKILL.md — the 4-backtick wrapping below is for plan display only)

````text
Audit all skill files for this project. Check both .claude/skills/ and skills/
directories (recursively) for skill definitions.

Apply your Skills Quality Rubric to each skill. Flag skills scoring below 35/50.

Output a structured report:
## Skills Inventory
## Per-Skill Evaluation (table: Dimension / Score / Notes)
## Priority Issues (must fix)
## Recommendations (improvements)

Do NOT apply any changes. Report only.
````

**code-quality-scouter** (script-offloading analysis): (same — write as ` ```text ` in SKILL.md)

````text
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
````

Capture and display both reports (skill-evaluator and scouter) before proceeding.

<!-- END REPLACEMENT TEXT -->

- [ ] **Step 3: Verify the edit**

Read the Phase 2 section of `skills/audit/SKILL.md` and confirm:

- The heading says "Phase 2 — Skills Quality & Efficiency Audit (skill-evaluator + code-quality-scouter)"
- The intro line says "launch **both** of the following subagents **in parallel**"
- The skill-evaluator prompt is inside a fenced code block under a **skill-evaluator:** bold label
- The scouter prompt is inside a fenced code block under a **code-quality-scouter** (script-offloading analysis): bold label
- The closing line reads "Capture and display both reports (skill-evaluator and scouter) before proceeding."
- The old closing line "Capture and display the full skills report before proceeding." is gone
- Phase 3 heading still follows after the `---` separator

- [ ] **Step 4: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "feat: add parallel scouter dispatch to Phase 2 for script-offloading analysis"
```

---

## Chunk 2: SKILL.md Phase 5 and 6 updates

### Task 2: Update Phase 5 template

**Files:**

- Modify: `skills/audit/SKILL.md` (Phase 5 section — locate by content, line numbers shifted after Task 1)

- [ ] **Step 1: Fix Phase 5 preamble**

Find:

```text
Synthesize all four reports into a single **Config Doctor Summary**:
```

Replace with:

```text
Synthesize all phase reports into a single **Config Doctor Summary**:
```

- [ ] **Step 2: Add Script Offloading Opportunities section to Phase 5 template**

Find (inside the Phase 5 fenced code block):

```text
### Memory Savings
[Projected token reduction if memory optimizations are applied. Include token savings from skill prompt compression.]

### Risk Assessment
```

Replace with:

```text
### Memory Savings
[Projected token reduction if memory optimizations are applied. Include token savings from skill prompt compression.]

### Script Offloading Opportunities
[Top opportunities from the scouter's analysis, ordered by estimated token savings. For each: skill name, opportunity description, category, estimated savings, effort level.]

### Risk Assessment
```

- [ ] **Step 3: Verify the edit**

Read the Phase 5 section and confirm:

- Preamble says "all phase reports"
- "Script Offloading Opportunities" appears between "Memory Savings" and "Risk Assessment"

- [ ] **Step 4: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "feat: add Script Offloading Opportunities to Phase 5 summary template"
```

---

### Task 3: Update Phase 6 option C

**Files:**

- Modify: `skills/audit/SKILL.md` (Phase 6 section — locate by content)

- [ ] **Step 1: Add script-offloading note to option C**

Find:

```text
- Rewrite skills scoring below 35/50 using skill-evaluator's suggested revisions

Ask the user which option they want before proceeding. Default to A if no response.
```

Replace with:

```text
- Rewrite skills scoring below 35/50 using skill-evaluator's suggested revisions
- Present script-offloading recommendations for acknowledgment (not auto-implemented — writing scripts is outside audit enforcement scope)

Ask the user which option they want before proceeding. Default to A if no response.
```

- [ ] **Step 2: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "feat: add script-offloading note to Phase 6 option C"
```

---

### Task 4: Update Phase name mapping table

**Files:**

- Modify: `skills/audit/SKILL.md` (User Arguments section — locate by content)

- [ ] **Step 1: Update the skills phase description in the mapping table**

Find:

```text
| `skills` | Phase 2 — Skills Quality Audit |
```

Replace with:

```text
| `skills` | Phase 2 — Skills Quality & Efficiency Audit |
```

- [ ] **Step 2: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "feat: update phase name mapping table for renamed Phase 2"
```

---

## Chunk 3: CLAUDE.md edit

### Task 5: Update CLAUDE.md repository structure comment

**Files:**

- Modify: `CLAUDE.md:19`

- [ ] **Step 1: Update the SKILL.md description line**

Find:

```text
  SKILL.md           # The full multi-phase audit orchestration logic (7 phases)
```

Replace with:

```text
  SKILL.md           # The full multi-phase audit orchestration logic (7 phases, dual Phase 2 dispatch)
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md to reflect dual Phase 2 dispatch"
```

---

## Chunk 4: Verification

### Task 6: Full read-through verification

- [ ] **Step 1: Read the complete SKILL.md**

Read `skills/audit/SKILL.md` from top to bottom and verify:

1. Phase 2 heading includes both agent names
2. Phase 2 dispatches two agents with correct prompts
3. Phase 2 closing line mentions both reports
4. Old closing line "Capture and display the full skills report before proceeding." is absent
5. Phase 5 preamble says "all phase reports" (not "four")
6. Phase 5 template includes "Script Offloading Opportunities" between "Memory Savings" and "Risk Assessment"
7. Phase 6 option C includes the script-offloading acknowledgment note
8. Phase name mapping table says "Skills Quality & Efficiency Audit"
9. No other content was accidentally damaged

- [ ] **Step 2: Read CLAUDE.md and verify the updated line**

- [ ] **Step 3: Run markdown lint**

```bash
npx markdownlint-cli2 skills/audit/SKILL.md CLAUDE.md
```

Fix any lint errors before proceeding.
