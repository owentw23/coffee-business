---
tags: [session, collaboration, handoff]
status: evergreen
created: 2026-09-28
modified: 2026-09-28
source: user-confirmed-project-context
---
# Shared Codex and Claude workspace — 2026-09-28

## Request
The project owner said the GitHub repository is `coffee-business` and Claude can access this project folder too.

## Verified state
- Local checkout: `/Users/owentw/Documents/coffee business`.
- Branch: `main`, tracking `origin/main`.
- Remote: `https://github.com/owentw23/coffee-business.git`.
- Working tree was clean before this update.
- User reports Claude Code can access the same folder; that access was not independently verified.

## Handoff decision
Use the shared Obsidian vault, `PROJECT_PLAN.md`, and `PROGRESS.md` as the common project context. Both assistant instruction files now ask agents to inspect Git status before work, preserve each other's uncommitted changes, avoid simultaneous edits to the same file, and coordinate through the owner and shared notes. This does not assume direct assistant-to-assistant messaging.

## Next action
Record the actual business goal, constraints, and success criteria in `PROJECT_PLAN.md`.
