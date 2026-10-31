# Repository working rules

## Isolate spec implementation

Before implementing a spec produced by brainstorming or picking up an approved
spec from `docs/superpowers/specs/`, create a new worktree and use a `codex/`
branch. Never implement in the user's checkout or on `main` unless the user
explicitly asks for that. Initializing a new repository is the other exception.

Prefer the harness's native worktree tool when available. Otherwise use
`.worktrees/<task>`; ensure `.worktrees/` is ignored before creating it. Keep
approved specs in `docs/superpowers/specs/` and implementation plans in
`docs/superpowers/plans/`.

A worktree's completed work must be committed, pushed, and delivered through a
pull request. If incomplete, push and open a draft PR explaining what remains.
Do not leave the only copy of work in a disposable local worktree. Keep an active
preview worktree until the user finishes trying it.

When handing over any PR, include bullets describing each substantive behavior,
what the PR intentionally does not do and why, decisions the user may want
reversed, and known gaps left unresolved. A bare PR link is insufficient.

## Browser verification

Use Microsoft Edge for interactive browser tests, local previews, and visual QA
in dedicated QA sessions. Use another browser only at the user's explicit
request. If Edge is unavailable, report that limitation instead of falling back.

## Keep long work observable

Before work likely to exceed 60 seconds, say what is running and the next
observable checkpoint. Use background/yielded commands, with blocking waits at
most 60 seconds. At least every 60 seconds, inspect real output or state and
report completed work, outstanding work, and blockers. After an interval with no
progress, investigate or change approach instead of repeating a blind wait.
For delegated work, request findings after the first quiet interval; after a
second interval without a deliverable, require a minimal result or take over.
Do not leave GUI startup or approval prompts unattended.

## Choose models deliberately

Only delegate when authorized by the user or an applicable skill. Set the model
and reasoning effort explicitly and record the reason. Prefer Luna/Terra for
bounded lookups and mechanical changes, Sol or stronger for substantive review
and integration, and the strongest appropriate available model for difficult
architecture, concurrency, or security decisions. Use Fable sparingly, only for
hard or high-stakes work or an explicit request. Do not silently equate model
names between providers or escalate routine tasks to an expensive tier.

## Local running and verification

`npm ci --ignore-scripts` installs the locked JavaScript dependencies. The normal
runtime is Bun (`bun start`). ZeroTTS uses `.venv/bin/python` when present; Python
3.12 is a practical local choice. This repo currently has no automated test suite.

`server.js` normally listens on all interfaces and auto-connects to
`@zenfacts_vn`. For an offline preview, use a localhost-only runtime launcher and
disable auto-connect before starting; do not expose the unauthenticated RPC or
admin interface. Live data in `data/merit-state.json` is tracked, so avoid
committing preview interactions, viewer data, audio caches, or Python environments.
