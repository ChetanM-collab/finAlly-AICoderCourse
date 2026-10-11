---
name: feature-dev
description: Use this to create a new feature branch when we introduce a new feature to this project
---

# Feature Branch

Create a fresh branch off an up-to-date `main` before starting work on a new feature.

## Steps

1. **Pick a name**: `feature/<short-kebab-name>` derived from the feature (e.g. `feature/portfolio-api`, `feature/watchlist-panel`). If `$ARGUMENTS` is given, use it as the short name. Ask the user only if the feature is unclear.
2. **Check the working tree**: run `git status --short`. If there are uncommitted changes, stop and ask the user whether to commit, stash, or carry them onto the new branch. Never discard them.
3. **Update main**:
   ```
   git checkout main
   git pull origin main
   ```
4. **Create the branch**:
   ```
   git checkout -b feature/<short-kebab-name>
   ```
   If the branch already exists, stop and ask whether to switch to it instead.
5. **Push and track**:
   ```
   git push -u origin feature/<short-kebab-name>
   ```
6. **Report** the branch name and confirm it tracks `origin/feature/<short-kebab-name>`.

## Rules

- `origin` is the user's repo; `upstream` is the course repo. Never push to `upstream`.
- Do not commit, merge, or open a PR as part of this skill. Feature work happens next, on the new branch.
