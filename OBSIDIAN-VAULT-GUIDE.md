# Coffee business Obsidian vault guide

Open the project’s `Knowledge/` folder in Obsidian with **Open folder as vault**. Start at `START-HERE.md`. Obsidian does not need to be open for a file-capable assistant to maintain the Markdown files; no Obsidian plugin or MCP server is required.

At the start of substantive work, read the vault entry point and rules, then the root `PROGRESS.md` and relevant `PROJECT_PLAN.md` stage. Preserve external originals under `Raw/`; put unfinished thoughts in `Inbox/`; keep decisions and session history in `Projects/`; store sourced reusable learning in `Resources/`; use `Areas/` and `Archive/` as needed. Automatic captures, if enabled, are drafts in `Inbox/Auto/`.

`PROJECT_PLAN.md` is the roadmap; `PROGRESS.md` is the live handoff; the vault records why and how. Keep one canonical owner per fact. For meaningful work, write an evidence-based checkpoint, update an existing note if possible, link it from a hub, append `Log.md`, review the Git diff, and commit only intended paths. The vault lives in the same Git repository as the project; a local commit is not an off-device backup until it is pushed to GitHub.

Run `node scripts/vault-lint.cjs` from the project root after structural changes, if Node.js is available. It checks structure and links; manually inspect semantic duplicates, unresolved contradictions, and claim accuracy.
