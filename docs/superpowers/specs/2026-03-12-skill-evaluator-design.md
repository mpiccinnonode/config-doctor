# Skill-Evaluator Agent Design

## Summary

Add a dedicated `skill-evaluator` agent to config-doctor that provides deep quality analysis of Claude Code skill files. Integrate it into the audit pipeline as a new Phase 2 (renumbering subsequent phases) and remove skills from agent-architect's scope.

## Decisions

- **Standalone agent** (`agents/skill-evaluator.md`) rather than extending agent-architect — clean ownership split
- **Sonnet model** — structural analysis doesn't need opus, saves tokens
- **Mirrored architecture** — parallels agent-architect's structure (Tool Usage, Role, Evaluation Mode, Rubric, Output Format, Self-Verification) with skills-specific content throughout
- **Rubric informed by Anthropic's skill-creator** — incorporates structural validation from [`quick_validate.py`](https://github.com/anthropics/skills/blob/main/skills/skill-creator/scripts/quick_validate.py), quality criteria from the [Grader/Comparator agents](https://github.com/anthropics/skills/tree/main/skills/skill-creator/agents), and best practices from `superpowers:writing-skills`
- **Phase renumbering** — new skills phase becomes Phase 2; old Phases 2/3/4/5 become 3/4/5/6

## Evaluation Rubric (5 dimensions, 0-10 each, /50)

### Frontmatter & Naming (0-10)

- Valid YAML with required fields (`name`, `description`)
- Kebab-case name, max 64 chars
- Description starts with "Use when...", describes only triggers (never workflow summary), max 1024 chars, no angle brackets
- Only allowed frontmatter keys (`name`, `description`, `allowed-tools`, `argument-hint`, `model`)
- `argument-hint` matches actual `$ARGUMENTS` parsing
- Version field present if a sibling `plugin.json` exists (detect by checking for `plugin.json` in the repo root)

### Prompt Architecture (0-10)

- Clear phase/step structure with logical sequencing
- Role statement present
- No ambiguous handoffs between phases
- Explicit success criteria per phase
- Output format expectations defined for each phase
- Escalation/fallback strategies for failures
- Genuine task completion instructions, not surface-level compliance

### Tool Scoping (0-10)

- `allowed-tools` is minimal and correct
- No over-broad wildcards unless justified (e.g., MCP namespace)
- No missing tools that the skill body references
- No tools listed that are never used
- Cross-references use explicit requirement markers, never force-loading patterns

### Argument Handling (0-10)

- `$ARGUMENTS` parsing covers all documented flags
- Sensible defaults when no args provided
- No undocumented flags
- Flag conflicts handled explicitly
- `argument-hint` in frontmatter matches the actual flag set

### Token Efficiency (0-10)

- Subagent prompts are concise, no redundant instructions
- Phase outputs aren't re-read unnecessarily
- Inline content that could be delegated to agents is flagged
- Soft benchmarks (from `superpowers:writing-skills` CSO guidelines): getting-started <150 words, frequently-loaded <200 words, complex orchestration <500 words per phase prompt. These are guidelines, not hard limits — deductions scale with how far over the target.
- Description doesn't duplicate workflow (prevents LLM shortcutting)

Scores below 35/50 trigger a "Suggested Revision" section in the output.

## Output Format

Markdown table (Dimension / Score / Notes) per skill evaluated, followed by:

- Strengths
- Issues Found
- Recommendations
- Revised Configuration (if score < 35/50)

## Skill-Evaluator Frontmatter

```yaml
---
name: skill-evaluator
description: "Use this agent when you need to evaluate existing skill files for quality,
  correctness, and adherence to best practices.\n\n<example>\nContext: User wants a
  skill reviewed.\nuser: \"Can you review my deploy skill and tell me if it's well-structured?\"\n
  assistant: \"I'll use the skill-evaluator to assess your skill's quality.\"\n<commentary>
  Use the Agent tool to launch skill-evaluator for skill quality review.</commentary>\n
  </example>\n\n<example>\nContext: Audit is evaluating project skills.\nuser: \"Run
  the config-doctor audit on my project.\"\nassistant: \"Phase 2 launches the skill-evaluator
  to assess all skill files.\"\n<commentary>The audit skill dispatches skill-evaluator
  as part of Phase 2.</commentary>\n</example>"
model: sonnet
---
```

## Audit Integration

### Phase Renumbering

The audit currently has Phases 0-5. After this change:

| Phase | Name | Agent | Before |
|-------|------|-------|--------|
| 0 | Orientation | (orchestrator) | Phase 0 |
| 1 | Agent & Configuration Quality | agent-architect | Phase 1 |
| **2** | **Skills Quality Audit** | **skill-evaluator** | **(new)** |
| 3 | Tooling Gap Analysis | code-quality-scouter | Phase 2 |
| 4 | Memory Optimization | memory-optimizer | Phase 3 |
| 5 | Consolidated Findings | (orchestrator) | Phase 4 |
| 6 | Enforcement Decision | (orchestrator) | Phase 5 |

### New Phase 2 prompt in `skills/audit/SKILL.md`

```text
## Phase 2 — Skills Quality Audit (skill-evaluator)

Use the Agent tool to launch the **skill-evaluator** subagent with the following prompt:

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

### New flag: `--skip-skills`

Skips Phase 2. Added to `$ARGUMENTS` documentation in both SKILL.md and CLAUDE.md.

### Agent-Architect Scope Change (exact edits)

In `agents/agent-architect.md`:

1. **Description field**: change `"evaluate, design, or refactor agent configurations, system prompts, rule sets, or skill definitions"` to `"evaluate, design, or refactor agent configurations, system prompts, and rule sets"`
2. **Evaluation Mode intro**: add line after "Your two primary modes of operation are:" section: `"**Scope:** Agent configurations, rule sets, and CLAUDE.md. Skills are evaluated by the dedicated skill-evaluator agent."`
3. **Creation Mode Step 5 (Scaffold Design)**: keep skills in scaffold output (agent-architect still creates scaffolds that include skills slots) but add note: `"For detailed skill evaluation, defer to the skill-evaluator agent."`
4. **Plugin Convention Awareness**: keep skill frontmatter documentation (needed for Creation Mode) but add: `"Skill quality evaluation is handled by the skill-evaluator agent, not this agent."`

In `skills/audit/SKILL.md` Phase 1 prompt:

5. Remove from Phase 1 prompt (line 43 of current SKILL.md): the clause `check both .claude/skills/ and skills/ for skills;` from the sentence `"Check both .claude/agents/ and agents/ directories for agent definitions; check both .claude/skills/ and skills/ for skills; check .claude/rules/ for rules files."`
6. Remove: `"## Skills Audit"` from the output format
7. Change Phase 1 heading to: `"## Phase 1 — Agent & Rules Quality Audit (agent-architect)"`

### Phase 5 (was Phase 4 — Consolidated Findings)

Add skills findings within existing cross-cutting categories (not a separate section):

- Critical Issues: include any skill scoring below 25/50
- High-Priority Improvements: include skills scoring 25-34/50
- Memory Savings: include token savings from skill prompt compression

### Phase 6 (was Phase 5 — Enforcement Decision)

- Option B (apply safe): Fix frontmatter issues in skills (missing fields, description format, naming violations)
- Option C (apply all): Rewrite skills scoring below 35/50 using skill-evaluator's suggested revisions

### Version field

The current SKILL.md frontmatter is missing the `version` field that CLAUDE.md says should be in sync with `plugin.json`. This is a pre-existing issue. Out of scope for this design — flag it as a separate fix. **Sequencing note:** the version field fix should land before or simultaneously with this work, otherwise the skill-evaluator will flag the audit skill itself on its first run.

## Files Changed

| File | Action | Specific Changes |
|------|--------|-----------------|
| `agents/skill-evaluator.md` | Create | New agent: Tool Usage (Serena preference, same as other agents), Role, Evaluation Mode with rubric, Output Format, Self-Verification Checklist |
| `skills/audit/SKILL.md` | Modify | Add Phase 2, renumber Phases 2-5 to 3-6, add `--skip-skills` flag, remove skills from Phase 1 prompt, update Phase 5/6 |
| `agents/agent-architect.md` | Modify | Update description (remove "skill definitions"), add scope boundary notes (4 locations listed above) |
| `CLAUDE.md` | Modify | Add `skill-evaluator.md` to agents listing, add `--skip-skills` to flags documentation, update phase count reference |

## References

- Anthropic [`skill-creator`](https://github.com/anthropics/skills/tree/main/skills/skill-creator): structural validation via `quick_validate.py`, Grader/Comparator agent rubrics
- `superpowers:writing-skills`: frontmatter rules, description best practices, token efficiency targets (CSO guidelines)
- Existing `agents/agent-architect.md`: mirrored structure pattern
