# Skill-Evaluator Agent Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a skill-evaluator agent to config-doctor and integrate it into the audit pipeline as Phase 2.

**Architecture:** New `agents/skill-evaluator.md` mirrors agent-architect's structure with skills-specific rubric. Audit skill gains Phase 2 (skills), renumbers subsequent phases, and agent-architect drops skills from its scope.

**Tech Stack:** Pure markdown — no build, no tests, no scripts. Verification via markdownlint.

**Spec:** `docs/superpowers/specs/2026-03-12-skill-evaluator-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `agents/skill-evaluator.md` | Create | Standalone skill evaluation agent with 5-dimension /50 rubric |
| `agents/agent-architect.md` | Modify | Remove skills from evaluation scope (4 edits) |
| `skills/audit/SKILL.md` | Modify | Add Phase 2, renumber phases, add `--skip-skills`, update Phase 1/5/6, fix Preflight cross-ref |
| `CLAUDE.md` | Modify | Add skill-evaluator to listing, add `--skip-skills` flag, update phase count and agent count |

---

## Chunk 1: Pre-existing Fix and New Agent

### Task 1: Fix missing version field in SKILL.md frontmatter

Per spec sequencing note: version field fix must land before or with this work.

**Files:**
- Modify: `skills/audit/SKILL.md:1-6` (frontmatter)

- [ ] **Step 1: Add version field to SKILL.md frontmatter**

Current frontmatter:

```yaml
---
name: audit
description: Deep-scan a project's Claude configuration...
argument-hint: "[--report-only | --apply-safe | --apply-all] [--skip-agents] [--skip-tooling] [--skip-memory] [/path/to/project]"
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash, Agent, "mcp__plugin_config-doctor_serena__*"]
---
```

Add `version: "1.0.1"` (matching `plugin.json`) after `name:`:

```yaml
---
name: audit
version: "1.0.1"
description: Deep-scan a project's Claude configuration...
argument-hint: "[--report-only | --apply-safe | --apply-all] [--skip-agents] [--skip-tooling] [--skip-memory] [/path/to/project]"
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash, Agent, "mcp__plugin_config-doctor_serena__*"]
---
```

- [ ] **Step 2: Verify markdown lint passes**

Run: `npx markdownlint-cli2 skills/audit/SKILL.md`
Expected: No errors (or only pre-existing ones unrelated to frontmatter)

- [ ] **Step 3: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "fix: add missing version field to audit skill frontmatter"
```

---

### Task 2: Create skill-evaluator agent

**Files:**
- Create: `agents/skill-evaluator.md`

- [ ] **Step 1: Create `agents/skill-evaluator.md`**

The file mirrors `agents/agent-architect.md` structure. Full content:

````markdown
---
name: skill-evaluator
description: "Use this agent when you need to evaluate existing skill files for quality, correctness, and adherence to best practices.\n\n<example>\nContext: User wants a skill reviewed.\nuser: \"Can you review my deploy skill and tell me if it's well-structured?\"\nassistant: \"I'll use the skill-evaluator to assess your skill's quality.\"\n<commentary>Use the Agent tool to launch skill-evaluator for skill quality review.</commentary>\n</example>\n\n<example>\nContext: Audit is evaluating project skills.\nuser: \"Run the config-doctor audit on my project.\"\nassistant: \"Phase 2 launches the skill-evaluator to assess all skill files.\"\n<commentary>The audit skill dispatches skill-evaluator as part of Phase 2.</commentary>\n</example>"
model: sonnet
---

## Tool Usage

Prefer Serena MCP tools for all file exploration. They return structured results at lower token cost than reading raw files.

| Task | Use this tool |
| ------ | -------------- |
| List a directory | `mcp__plugin_config-doctor_serena__list_dir` |
| Read a file | `mcp__plugin_config-doctor_serena__read_file` |
| Search content across files | `mcp__plugin_config-doctor_serena__search_for_pattern` |
| Find a file by name | `mcp__plugin_config-doctor_serena__find_file` |
| Get file structure without full read | `mcp__plugin_config-doctor_serena__get_symbols_overview` |

Fall back to `Read`, `Glob`, or `Grep` only if a Serena tool is unavailable or returns an error.

---

You are a Skill Quality Analyst — an expert in evaluating Claude Code skill files for structural correctness, prompt quality, and operational efficiency. You understand skill frontmatter contracts, argument handling patterns, tool scoping, and the principles that make skills reliable and token-efficient.

Your two primary modes of operation are:

1. **Evaluation Mode**: Assess existing skill files for quality using the Skills Quality Rubric below.
2. **Creation Mode**: Design new skill configurations with correct frontmatter, clear prompt architecture, and minimal tool scoping.

**Scope:** Skill definition files (SKILL.md and similar). Agent configurations and rule sets are evaluated by the agent-architect agent, not this agent.

---

## Evaluation Mode

When asked to evaluate skill files, apply the following rubric to each skill found:

### Skills Quality Rubric

#### Frontmatter & Naming (0-10)

- Valid YAML with required fields (`name`, `description`)
- Kebab-case name, max 64 characters
- Description starts with "Use when...", describes only triggering conditions (never workflow summary), max 1024 characters, no angle brackets
- Only allowed frontmatter keys (`name`, `description`, `allowed-tools`, `argument-hint`, `model`, `version`)
- `argument-hint` matches actual `$ARGUMENTS` parsing in the skill body
- Version field present if a `plugin.json` exists in the repository root (detect by checking for `plugin.json`)

#### Prompt Architecture (0-10)

- Clear phase/step structure with logical sequencing
- Role statement or orchestration context present
- No ambiguous handoffs between phases or subagent calls
- Explicit success criteria per phase
- Output format expectations defined for each phase
- Escalation/fallback strategies for failure cases
- Instructions target genuine task completion, not surface-level compliance

#### Tool Scoping (0-10)

- `allowed-tools` list is minimal and correct for what the skill body actually uses
- No over-broad wildcards unless justified (e.g., MCP namespace wildcard for a known server)
- No missing tools that the skill body references or implies
- No tools listed that are never used in the skill body
- Cross-references to other skills or agents use explicit requirement markers, never force-loading patterns

#### Argument Handling (0-10)

- `$ARGUMENTS` parsing covers all flags documented in the `argument-hint`
- Sensible defaults when no arguments are provided
- No undocumented flags accepted in the parsing logic
- Flag conflicts are handled explicitly (e.g., `--report-only` vs `--apply-all`)
- The `argument-hint` string in frontmatter matches the actual set of supported flags

#### Token Efficiency (0-10)

- Subagent prompts are concise with no redundant instructions
- Phase outputs are not re-read or re-summarized unnecessarily
- Inline content that could be delegated to a subagent is flagged
- Soft benchmarks: getting-started workflows target <150 words per prompt, frequently-loaded skills <200 words overhead, complex orchestration <500 words per phase prompt (guidelines, not hard limits — deductions scale with how far over target)
- Description field does not duplicate the skill's workflow (this causes LLMs to follow the description as a shortcut instead of reading the full skill)

### Evaluation Output Format

For each skill evaluated, present:

| Dimension | Score | Notes |
|-----------|-------|-------|
| Frontmatter & Naming | X/10 | ... |
| Prompt Architecture | X/10 | ... |
| Tool Scoping | X/10 | ... |
| Argument Handling | X/10 | ... |
| Token Efficiency | X/10 | ... |
| **Total** | **X/50** | |

Follow with:

- **Strengths**: what the skill does well
- **Issues Found**: specific problems with line references where possible
- **Recommendations**: prioritized improvements
- **Suggested Revision**: (only if total score < 35/50) a revised version of the skill or the most critical sections

---

## Creation Mode

When asked to create a new skill, follow this methodology:

### Step 1: Requirements Extraction

Before creating anything, identify:

- **Core purpose**: What task does this skill orchestrate?
- **Triggering conditions**: When should this skill be invoked?
- **Tool requirements**: What tools does the skill need access to?
- **Arguments**: What flags or parameters should users be able to pass?
- **Subagents**: Does the skill delegate work to agents? Which ones?
- **Project context**: What conventions or patterns are in use?

If any of these are unclear, ask targeted clarifying questions before proceeding.

### Step 2: Frontmatter Design

- `name`: kebab-case, 2-4 words, max 64 characters
- `description`: starts with "Use when...", max 1024 characters, triggers only (never workflow)
- `allowed-tools`: minimal set — only tools the skill body actually uses
- `argument-hint`: matches the `$ARGUMENTS` parsing exactly
- `model`: only if the skill requires a specific model

### Step 3: Prompt Architecture

Structure the skill body with:

1. **Context/role statement** — what the skill orchestrates
2. **Phases or steps** — logical units of work with clear sequencing
3. **Subagent prompts** — concise delegation with explicit output format expectations
4. **Output expectations** — what the skill produces at the end
5. **Argument parsing** — `$ARGUMENTS` section at the end with flag handling

### Step 4: Verification

Before delivering, verify:

- Every `argument-hint` flag has corresponding parsing logic
- Every tool in `allowed-tools` is actually used
- Every subagent prompt has an explicit output format
- Description doesn't summarize the workflow
- Token efficiency targets are met

---

## Cross-Cutting Principles

**Structural correctness over style**: A skill with perfect formatting but broken argument handling is worse than an ugly skill that works correctly. Prioritize functional correctness.

**Minimal tool surface**: Every tool in `allowed-tools` is an attack surface and a token cost. If a skill doesn't use `Write`, don't include it. If a skill only reads files, it doesn't need `Bash`.

**Description is a search index, not documentation**: The description field exists to help Claude decide whether to load the skill. It must answer "should I read this right now?" — not "what does this skill do step by step?"

**Argument-hint is a contract**: If the hint says `--skip-agents` is supported, the skill body must handle it. If the skill body accepts `--verbose`, the hint must list it.

---

## Plugin Convention Awareness (config-doctor)

When operating within the config-doctor plugin repository, apply these additional standards:

- **Skill frontmatter**: requires `name:`, `description:`, `version:`, and `allowed-tools:` fields
- **`$ARGUMENTS` contract**: the audit skill passes user flags via `$ARGUMENTS` at the end of the skill file
- **Version sync**: `plugin.json` and `SKILL.md` both declare a `version` field and must be kept in sync
- **No code files**: this plugin is pure markdown. Never suggest scripts or compiled assets for skill logic

Skill quality evaluation is the responsibility of this agent. Agent and rule quality evaluation is handled by the agent-architect agent.

---

## Self-Verification Checklist

Before delivering any output, verify:

- Does each evaluated skill have scores for all five rubric dimensions?
- Are issues referenced with specific line numbers or section names?
- Does the frontmatter of each skill have all required fields?
- Does every `$ARGUMENTS` flag have documented parsing behavior?
- Are `allowed-tools` minimal — no unused tools listed?
- Does the description avoid summarizing the skill's workflow?
- Is the `argument-hint` consistent with the actual flag set?
- For suggested revisions: is the revision complete and self-contained?
- Would a capable LLM reading this evaluation know exactly what to fix?

If any check fails, revise before delivering output.
````

- [ ] **Step 2: Verify markdown lint passes**

Run: `npx markdownlint-cli2 agents/skill-evaluator.md`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add agents/skill-evaluator.md
git commit -m "feat: add skill-evaluator agent with 5-dimension quality rubric"
```

---

## Chunk 2: Agent-Architect Scope Change

### Task 3: Remove skills from agent-architect evaluation scope

**Files:**
- Modify: `agents/agent-architect.md:3` (description field)
- Modify: `agents/agent-architect.md:23-29` (Evaluation Mode intro)
- Modify: `agents/agent-architect.md:114-144` (Creation Mode Step 5)
- Modify: `agents/agent-architect.md:171-181` (Plugin Convention Awareness)

- [ ] **Step 1: Update description field**

Change line 3 — in the first sentence of the description string, replace:

```text
evaluate, design, or refactor agent configurations, system prompts, rule sets, or skill definitions
```

with:

```text
evaluate, design, or refactor agent configurations, system prompts, and rule sets
```

- [ ] **Step 2: Add scope boundary to Evaluation Mode**

After line 29 (the closing of the two modes list), add:

```markdown
**Scope:** Agent configurations, rule sets, and CLAUDE.md. Skills are evaluated by the dedicated skill-evaluator agent.
```

- [ ] **Step 3: Add note to Creation Mode Step 5 (Scaffold Design)**

After the scaffold example closing ``` (around line 144), add:

```markdown
For detailed skill evaluation, defer to the skill-evaluator agent.
```

- [ ] **Step 4: Add note to Plugin Convention Awareness**

After the last bullet in the Plugin Convention Awareness section (around line 179), add:

```markdown
- **Skill evaluation boundary**: Skill quality evaluation is handled by the skill-evaluator agent, not this agent. This agent retains skill frontmatter knowledge for Creation Mode scaffolding.
```

- [ ] **Step 5: Verify markdown lint passes**

Run: `npx markdownlint-cli2 agents/agent-architect.md`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add agents/agent-architect.md
git commit -m "refactor: remove skills from agent-architect evaluation scope"
```

---

## Chunk 3: Audit Skill Integration

### Task 4: Update audit skill with Phase 2 and renumbering

This is the largest task. Apply changes top-to-bottom through the file.

**Files:**
- Modify: `skills/audit/SKILL.md:1-6` (frontmatter — argument-hint)
- Modify: `skills/audit/SKILL.md:20` (Preflight cross-ref)
- Modify: `skills/audit/SKILL.md:38` (Phase 1 heading)
- Modify: `skills/audit/SKILL.md:43` (Phase 1 prompt — remove skills clause)
- Modify: `skills/audit/SKILL.md:49-56` (Phase 1 output — remove Skills Audit)
- Insert: after Phase 1 block (after line 58) — new Phase 2
- Modify: all subsequent phase headings (renumber 2→3, 3→4, 4→5, 5→6)
- Modify: `skills/audit/SKILL.md:118` (Phase 4 "three reports" → "four reports")
- Modify: `skills/audit/SKILL.md:178-186` (User Arguments — add --skip-skills)

- [ ] **Step 1: Update argument-hint in frontmatter**

Change line 4:

```yaml
argument-hint: "[--report-only | --apply-safe | --apply-all] [--skip-agents] [--skip-tooling] [--skip-memory] [/path/to/project]"
```

to:

```yaml
argument-hint: "[--report-only | --apply-safe | --apply-all] [--skip-agents] [--skip-skills] [--skip-tooling] [--skip-memory] [/path/to/project]"
```

- [ ] **Step 2: Fix Preflight cross-reference**

Change line 20 — replace `Phase 4 summary` with `Phase 5 summary`:

```text
- **Declined or install failed:** proceed with native tools (`Read`, `Glob`, `Grep`) and note Serena's absence in the Phase 5 summary.
```

- [ ] **Step 3: Update Phase 1 heading**

Change line 38:

```markdown
## Phase 1 — Agent & Configuration Quality Audit (agent-architect)
```

to:

```markdown
## Phase 1 — Agent & Rules Quality Audit (agent-architect)
```

- [ ] **Step 4: Remove skills clause from Phase 1 prompt**

Change line 43:

```text
Check both .claude/agents/ and agents/ directories for agent definitions; check both .claude/skills/ and skills/ for skills; check .claude/rules/ for rules files.
```

to:

```text
Check both .claude/agents/ and agents/ directories for agent definitions; check .claude/rules/ for rules files. Skills are evaluated separately by the skill-evaluator agent.
```

- [ ] **Step 5: Remove Skills Audit from Phase 1 output format**

Remove the line `## Skills Audit` from the Phase 1 prompt output section (line 51 of the current file). The surrounding lines are `## Rules Audit` (line 50) and `## Priority Issues (must fix)` (line 52) — after removal, those two lines become adjacent.

- [ ] **Step 6: Insert new Phase 2 block**

After the Phase 1 closing `---` (line 60 of the current file — the `---` separator after `Capture and display the full report from agent-architect before proceeding.` on line 58), insert:

```markdown

## Phase 2 — Skills Quality Audit (skill-evaluator)

Use the Agent tool to launch the **skill-evaluator** subagent with the following prompt:

\```text
Audit all skill files for this project. Check both .claude/skills/ and skills/
directories (recursively) for skill definitions.

Apply your Skills Quality Rubric to each skill. Flag skills scoring below 35/50.

Output a structured report:
## Skills Inventory
## Per-Skill Evaluation (table: Dimension / Score / Notes)
## Priority Issues (must fix)
## Recommendations (improvements)

Do NOT apply any changes. Report only.
\```

Capture and display the full skills report before proceeding.

---
```

- [ ] **Step 7: Renumber Phase 2 → Phase 3**

Change heading:

```markdown
## Phase 2 — Tooling Gap Analysis (code-quality-scouter)
```

to:

```markdown
## Phase 3 — Tooling Gap Analysis (code-quality-scouter)
```

- [ ] **Step 8: Renumber Phase 3 → Phase 4**

Change heading:

```markdown
## Phase 3 — Memory Optimization
```

to:

```markdown
## Phase 4 — Memory Optimization
```

- [ ] **Step 9: Renumber Phase 4 → Phase 5 and update report count**

Change heading:

```markdown
## Phase 4 — Consolidated Findings
```

to:

```markdown
## Phase 5 — Consolidated Findings
```

Change `Synthesize all three reports` to `Synthesize all four reports`.

- [ ] **Step 10: Add skills findings categories to Phase 5 template**

In the Consolidated Findings template, under `### Critical Issues (must fix)`, add context that skills scoring below 25/50 belong here. Under `### High-Priority Improvements`, note skills scoring 25-34/50. Under `### Memory Savings`, note token savings from skill prompt compression.

This is guidance to the orchestrator — replace the existing bracketed placeholder text after each relevant heading. The current file has these placeholders:

- Line ~123: `[Issues that will cause agent misbehavior, contradictions, or significant token waste]`
- Line ~126: `[Impactful changes that are low-risk]`
- Line ~132: `[Projected token reduction if memory optimizations are applied]`

Replace each with the expanded version below:

```text
### Critical Issues (must fix)
[Issues that will cause agent misbehavior, contradictions, or significant token waste. Include skills scoring below 25/50.]
```

```text
### High-Priority Improvements
[Impactful changes that are low-risk. Include skills scoring 25-34/50.]
```

```text
### Memory Savings
[Projected token reduction if memory optimizations are applied. Include token savings from skill prompt compression.]
```

- [ ] **Step 11: Renumber Phase 5 → Phase 6**

Change heading:

```markdown
## Phase 5 — Enforcement Decision
```

to:

```markdown
## Phase 6 — Enforcement Decision
```

- [ ] **Step 12: Update enforcement options for skills**

In the Phase 6 Option B bullet list (currently 4 bullets starting with "Remove exact-duplicate content"), add as the last bullet:

```markdown
- Fix frontmatter issues in skill files (missing fields, description format, naming violations)
```

In the Phase 6 Option C description (currently one line: "Apply all findings including structural changes..."), add after "Requires explicit user confirmation before each significant change.":

```markdown
- Rewrite skills scoring below 35/50 using skill-evaluator's suggested revisions
```

- [ ] **Step 13: Update Enforcement Guidelines phase reference**

The Enforcement Guidelines section (lines ~163-173 of the current file) contains no phase number references — only references to "options B and C" and agent names. No changes needed in this section.

- [ ] **Step 14: Add --skip-skills to User Arguments**

In the User Arguments section, add after `--skip-agents`:

```markdown
- `--skip-skills` → Skip Phase 2
```

Update the existing skip flags to reference new phase numbers:

```markdown
- `--skip-agents` → Skip Phase 1
- `--skip-skills` → Skip Phase 2
- `--skip-tooling` → Skip Phase 3
- `--skip-memory` → Skip Phase 4
```

- [ ] **Step 15: Verify markdown lint passes**

Run: `npx markdownlint-cli2 skills/audit/SKILL.md`
Expected: No errors

- [ ] **Step 16: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "feat: add Phase 2 skills evaluation and renumber audit phases"
```

---

## Chunk 4: Documentation Update

### Task 5: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md:12-15` (agents listing)
- Modify: `CLAUDE.md:18` (phase count)
- Modify: `CLAUDE.md:31` (agent count)
- Modify: `CLAUDE.md:39` (flags list)

- [ ] **Step 1: Add skill-evaluator to agents listing**

After line 15 (`memory-optimizer.md`), add:

```text
  skill-evaluator.md      # Evaluates skill files for quality via a 0-50 rubric (sonnet)
```

- [ ] **Step 2: Update phase count**

Change line 18:

```text
  SKILL.md           # The full multi-phase audit orchestration logic (5 phases)
```

to:

```text
  SKILL.md           # The full multi-phase audit orchestration logic (7 phases)
```

- [ ] **Step 3: Update agent count in Serena section**

Change line 31:

```text
All three subagents are instructed to prefer **Serena MCP tools**
```

to:

```text
All four subagents are instructed to prefer **Serena MCP tools**
```

- [ ] **Step 4: Add --skip-skills to flags documentation**

Change line 39:

```text
- **`$ARGUMENTS`**: the skill receives user flags (`--report-only`, `--apply-safe`, `--apply-all`, `--skip-tooling`, `--skip-memory`, `--skip-agents`, or a path) via the `$ARGUMENTS` placeholder at the end of `SKILL.md`.
```

to:

```text
- **`$ARGUMENTS`**: the skill receives user flags (`--report-only`, `--apply-safe`, `--apply-all`, `--skip-agents`, `--skip-skills`, `--skip-tooling`, `--skip-memory`, or a path) via the `$ARGUMENTS` placeholder at the end of `SKILL.md`.
```

- [ ] **Step 5: Verify markdown lint passes**

Run: `npx markdownlint-cli2 CLAUDE.md`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add skill-evaluator agent and --skip-skills flag to CLAUDE.md"
```

---

## Chunk 5: Final Verification

### Task 6: Cross-file consistency check

- [ ] **Step 1: Verify all files lint clean**

Run: `npx markdownlint-cli2 agents/skill-evaluator.md agents/agent-architect.md skills/audit/SKILL.md CLAUDE.md`
Expected: No errors

- [ ] **Step 2: Verify flag consistency**

Check that `--skip-skills` appears in all three locations:
1. `skills/audit/SKILL.md` frontmatter `argument-hint`
2. `skills/audit/SKILL.md` User Arguments section
3. `CLAUDE.md` `$ARGUMENTS` documentation

Run: `grep -n "skip-skills" skills/audit/SKILL.md CLAUDE.md`
Expected: 3 matches (2 in SKILL.md, 1 in CLAUDE.md)

- [ ] **Step 3: Verify phase numbering is sequential**

Run: `grep -n "^## Phase" skills/audit/SKILL.md`
Expected: Phase 0, Phase 1, Phase 2, Phase 3, Phase 4, Phase 5, Phase 6 (7 headings, sequential)

- [ ] **Step 4: Verify skill-evaluator frontmatter matches spec**

Check that `agents/skill-evaluator.md` frontmatter fields (`name`, `description`, `model`) match the spec at `docs/superpowers/specs/2026-03-12-skill-evaluator-design.md` lines 73-87.

Run: `head -5 agents/skill-evaluator.md`
Expected: `name: skill-evaluator`, `model: sonnet`

- [ ] **Step 5: Verify agent-architect no longer references skill evaluation**

Run: `grep -n "skill" agents/agent-architect.md`
Verify: no lines suggest agent-architect should evaluate skills (Creation Mode references and the boundary note are expected)

- [ ] **Step 6: Verify version sync**

Check that `skills/audit/SKILL.md` frontmatter version matches `plugin.json` version (both should be `1.0.1`).

- [ ] **Step 7: Final commit (if any lint fixes needed)**

```bash
git add -A
git commit -m "fix: lint and consistency fixes for skill-evaluator integration"
```

Only run if Steps 1-6 revealed issues that needed fixing.
