# AGENTS.md — dep-sentry

This repo follows Alex Perreira’s machine-wide agent defaults. Additions below are project-scoped.

## Project Info
- Owner: Alex Perreira (`@alexperreira`)
- Site/blog: alexhacks.net
- Tech stack (seed): python, fastapi, sqlalchemy, alembic

## Autonomy
- Read-only inspection is allowed without approval.
- Anything that changes state (file edits, installs, generators, git push) requires explicit approval unless Alex explicitly asks for that action.

## Git Policy
- Inherits machine-wide defaults (branch + push for continuity; PR required for higher-risk changes).
- Add any repo-specific overrides here.

## Invariants (fill in)
- (Add rules that must not be broken without explicit approval.)

## Risk Zones (fill in)
- (List directories/files that require extra care.)

## Repo Quickstart (fill in as you go)

### Python (suggested)
- Create venv: `python -m venv .venv`
- Activate: `source .venv/bin/activate`
- Install: `pip install -r requirements.txt` (or `uv sync` if using uv)
- Lint: `ruff check .` (or your tool)
- Test: `pytest -q`

### Notes
- Prefer keeping the repo on the WSL filesystem (not under `/mnt/c`) for performance.
- Update this file with concrete commands once the stack stabilizes.
