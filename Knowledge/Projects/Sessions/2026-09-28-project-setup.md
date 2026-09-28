---
tags: [session, setup]
status: evergreen
created: 2026-09-28
modified: 2026-09-28
source: project-setup
---
# Project setup — 2026-09-28

## Request
Set up Obsidian memory for the Coffee business project, then install ECC skills locally.

## Decisions
- Adopted the starter vault structure and its evidence-based memory workflow.
- Added the ECC skills from the official upstream repository into `.agents/skills/` for this project.
- Kept ECC hooks, MCP configuration, plugins, global config, agents, and commands out of scope.

## Files and evidence
- Project memory: `Knowledge/START-HERE.md`, `Knowledge/VAULT-RULES.md`, `PROJECT_PLAN.md`, and `PROGRESS.md`.
- ECC provenance and scope: `ECC-SKILLS.md`; upstream commit `d3b8a3e908904e242ed2dbe66af62cca71131419`.
- 39 skill folders copied from the upstream `.agents/skills/` directory.

## Checks and limits
- Verified local skill folder count against the upstream checkout. No project implementation or business outcome has been verified.
- Obsidian has not been opened to confirm vault rendering.

## Next action
Provide the business goal, scope, constraints, and success criteria to fill in `PROJECT_PLAN.md`.
