---
name: commit
description: Analyze changes and create a well-formatted git commit
disable-model-invocation: true
allowed-tools: Bash(git *)
argument-hint: <optional commit intent description>
---

# Git Commit Skill

## Current Repository State

**Changed files:**
!`git status --short`

**Staged changes:**
!`git diff --cached`

**Unstaged changes:**
!`git diff`

**Recent commits (for style reference):**
!`git log --oneline -5`

## Instructions

1. Analyze all changes above (both staged and unstaged).
2. If there are unstaged changes, ask the user whether they want to stage them (all or selectively) before committing.
3. If there are no changes at all, inform the user and stop.
4. Generate a commit message in **Conventional Commits** format:
   - Type: `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`, `style`
   - Optional scope in parentheses: `feat(gateway): ...`
   - Subject line max 72 characters, imperative mood, no trailing period
   - Add a body separated by a blank line if the change needs more explanation
5. If the user provided arguments via `$ARGUMENTS`, use that as a hint for the commit intent.
6. Present the proposed commit message to the user and wait for confirmation before executing.
7. After committing, show the result with `git log --oneline -1`.
