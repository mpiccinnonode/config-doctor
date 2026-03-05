---
description: Deep-scan a project's Claude configuration (.claude/ directory, CLAUDE.md, agents, rules, skills, memory files). Produces a quality report, tooling gap analysis, and memory optimization recommendations — then optionally enforces significant changes.
argument-hint: [--report-only | --apply-safe | --apply-all] [--skip-tooling] [--skip-memory] [/path/to/project]
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash, Agent, "mcp__plugin_config-doctor_serena__*"]
---

Run the config-doctor skill.

$ARGUMENTS
