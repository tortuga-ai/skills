---
name: tortuga:update-claudemd
description: Update CLAUDE.md from latest git commits or full codebase scan. Keeps AI agent context in sync with code changes.
license: MIT
metadata:
  author: tortuga-ai
  version: "1.0.0"
  organization: Tortuga AI
  public: true
  repo: https://github.com/tortuga-ai/skills
  date: March 2026
  abstract: Reads recent git commits or the full codebase, then rewrites CLAUDE.md to reflect current project structure, commands, and architecture. Shows a diff before writing so the user can confirm.
---

Update CLAUDE.md to reflect the latest code changes.

## Usage

```
/update-claudemd                   # last 10 commits
/update-claudemd --commits 20      # last N commits
/update-claudemd --since v1.2.0    # since a tag or date
/update-claudemd --all             # full codebase scan
```

## Step 1 — Parse arguments and determine scope

- `--all` → scan the full codebase
- `--since <ref>` → git changes since a tag, branch, or date (e.g. `v1.0`, `main`, `2026-01-01`)
- `--commits N` → last N commits (default: 10)

## Step 2 — Gather context

### Commit-based (default):

```bash
git log --oneline -20
git diff --name-only HEAD~10 | grep -v node_modules | grep -v ".lock" | head -40
git diff HEAD~10 --stat
```

Read the changed source files (skip binaries and lock files, cap at ~20 files).

### Full codebase (`--all`):

```bash
find . -maxdepth 3 -not -path "*/node_modules/*" -not -path "*/.git/*" \
  -not -path "*/dist/*" -not -path "*/.next/*" -not -path "*/build/*" | sort | head -100
```

Read key manifest files (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`) and top-level source files.

## Step 3 — Read the existing CLAUDE.md

Find and read the CLAUDE.md:

```bash
find . -maxdepth 3 -not -path "*/node_modules/*" -not -path "*/.git/*" -name "CLAUDE.md" | sort
```

If no CLAUDE.md exists, ask the user before creating one from scratch.

## Step 4 — Generate the update

Apply surgical edits only — preserve hand-written and opinionated content. Only update factual, auto-derivable sections:

- **Project Structure** — update if files/directories were added or removed
- **Key Commands** — update if `package.json` scripts, Makefile targets, or CLI commands changed
- **Architecture** — update if significant new modules, patterns, or dependencies were introduced
- **Recent Changes** — add or update a section with up to 5 bullet points summarizing the latest commits; if the section already exists, replace it

Never remove content that looks hand-written or contains custom instructions. Do not add new sections unless the user already has them (except **Recent Changes**).

## Step 5 — Show diff and confirm

Show a before/after diff of the proposed changes:

```
──────────────────────────────────────────
  CLAUDE.md  (6 lines changed)
──────────────────────────────────────────
+ ## Recent Changes
+ - Added OAuth2 middleware (src/auth/oauth.ts)
+ - Renamed `startServer` → `bootstrap` in src/index.ts
+ - New env var: OAUTH_CLIENT_ID
```

Then ask: **"Write these changes to CLAUDE.md? (yes / no)"**

## Step 6 — Write the file

If confirmed, write the updated CLAUDE.md using the Edit tool for surgical changes where possible, or Write for a full rewrite if the structure changed significantly.

Also stamp a `<!-- last updated: YYYY-MM-DD -->` comment on the very first line (or update it if already present).

```
Done — CLAUDE.md updated (6 lines)
```
