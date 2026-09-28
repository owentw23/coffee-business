# Coffee business

Planning and working repository for a home-based coffee business. It includes the project's shared memory: an Obsidian vault that people and AI assistants both read and update.

## Getting started

```bash
git clone https://github.com/owentw23/coffee-business.git
```

1. Read [PROJECT_PLAN.md](PROJECT_PLAN.md) for the roadmap and [PROGRESS.md](PROGRESS.md) for current status and the next action.
2. Optional: open the `Knowledge/` folder in Obsidian with **Open folder as vault**. Start at `START-HERE.md`. See [OBSIDIAN-VAULT-GUIDE.md](OBSIDIAN-VAULT-GUIDE.md).
3. If you use Claude Code or Codex, it follows [CLAUDE.md](CLAUDE.md) or [AGENTS.md](AGENTS.md) automatically and reads the memory before working.

## Layout

| Path | Purpose |
|---|---|
| `PROJECT_PLAN.md` | Roadmap, stages, and acceptance gates |
| `PROGRESS.md` | Current state and handoff |
| `Knowledge/` | Obsidian vault: decisions, sources, session notes, log |
| `.agents/skills/` | Project-local ECC skills (see [ECC-SKILLS.md](ECC-SKILLS.md)) |
| `scripts/` | `vault-lint.cjs` checks vault structure; `vault-hook.cjs` is an optional capture hook |

## Working together

- Pull before you start: `git pull`.
- Commit your memory updates (`PROGRESS.md`, `Knowledge/`) together with the work they describe, then push.
- Don't commit secrets. `.env` files are ignored.
- After structural vault changes, run `node scripts/vault-lint.cjs`.
