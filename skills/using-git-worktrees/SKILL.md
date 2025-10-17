---
name: Using-Git-Worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

**Announce at start:** "I'm using the Using Git Worktrees skill to set up an isolated workspace."

## Shopify Monorepo (shop/world)

**When working in the shop/world monorepo, use `dev tree` commands instead of manual git worktree.**

The dev tool handles sparse checkout and zone management automatically.

### Creating a Worktree in the Monorepo

```bash
# Create worktree on branch "feature-name"
/opt/dev/bin/dev tree add feature-name

# Create and immediately switch to it
/opt/dev/bin/dev tree add feature-name -s
```

**What this does:**
1. Creates worktree at `~/world/trees/feature-name/`
2. Checks out branch `feature-name`
3. Sets up sparse checkout (only materializes zones you navigate to)

### Navigating Zones in a Worktree

```bash
# Navigate to a zone in specific worktree
/opt/dev/bin/dev cd //zone-name -t worktree-name

# Navigate to zone in current worktree
/opt/dev/bin/dev cd //zone-name -t .

# Navigate to zone in main worktree (default)
/opt/dev/bin/dev cd //zone-name
```

**Examples:**
```bash
# Main worktree
/opt/dev/bin/dev cd //admin
# → ~/world/trees/root/src//admin

# Feature worktree
/opt/dev/bin/dev tree add my-feature -s
/opt/dev/bin/dev cd //admin -t my-feature
# → ~/world/trees/my-feature/src//admin
```

### Managing Worktrees

```bash
# List all worktrees
/opt/dev/bin/dev tree list

# Show current worktree info
/opt/dev/bin/dev tree show

# Switch to existing worktree
/opt/dev/bin/dev tree switch worktree-name

# Remove worktree
/opt/dev/bin/dev tree remove worktree-name
```

### Project Setup in Monorepo Worktrees

After creating a worktree and navigating to a zone:

```bash
# Run from within the zone directory
/opt/dev/bin/dev up
```

This automatically:
- Sets up zone dependencies
- Runs necessary installations
- Configures the environment

### Verification

```bash
# Run tests for the zone
/opt/dev/bin/dev test

# Check coverage
/opt/dev/bin/dev coverage --include-branch-commits
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

### Quick Reference (Monorepo)

| Task | Command |
|------|---------|
| Create worktree | `dev tree add feature-name` |
| Create + switch | `dev tree add feature-name -s` |
| Navigate to zone | `dev cd //zone -t worktree-name` |
| List worktrees | `dev tree list` |
| Setup zone | `dev up` (from zone directory) |
| Run tests | `dev test` |

---

## Standalone Repositories

For non-monorepo projects, use manual git worktree commands.

## Directory Selection Process

Follow this priority order:

### 1. Check Existing Directories

```bash
# Check in priority order
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

### 2. Check CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**If preference specified:** Use it without asking.

### 3. Ask User

If no directory exists and no CLAUDE.md preference:

```
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## Safety Verification

### For Project-Local Directories (.worktrees or worktrees)

**MUST verify .gitignore before creating worktree:**

```bash
# Check if directory pattern in .gitignore
grep -q "^\.worktrees/$" .gitignore || grep -q "^worktrees/$" .gitignore
```

**If NOT in .gitignore:**

Per the rule "Fix broken things immediately":
1. Add appropriate line to .gitignore
2. Commit the change
3. Proceed with worktree creation

**Why critical:** Prevents accidentally committing worktree contents to repository.

### For Global Directory (~/.config/superpowers/worktrees)

No .gitignore verification needed - outside project entirely.

## Creation Steps

### 1. Detect Project Name

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. Create Worktree

```bash
# Determine full path
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# Create worktree with new branch
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. Run Project Setup

Auto-detect and run appropriate setup:

```bash
# Shopify (dev.yml indicates Shopify project)
if [ -f dev.yml ]; then /opt/dev/bin/dev up; fi

# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. Verify Clean Baseline

Run tests to ensure worktree starts clean:

```bash
# Examples - use project-appropriate command
/opt/dev/bin/dev test  # Shopify projects
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| `.worktrees/` exists | Use it (verify .gitignore) |
| `worktrees/` exists | Use it (verify .gitignore) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check CLAUDE.md → Ask user |
| Directory not in .gitignore | Add it immediately + commit |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

**Skipping .gitignore verification**
- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always grep .gitignore before creating project-local worktree

**Assuming directory location**
- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > CLAUDE.md > ask

**Proceeding with failing tests**
- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

**Hardcoding setup commands**
- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflow

```
You: I'm using the Using Git Worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify .gitignore - contains .worktrees/]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flags

**Never:**
- Create worktree without .gitignore verification (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume directory location when ambiguous
- Skip CLAUDE.md check

**Always:**
- Follow directory priority: existing > CLAUDE.md > ask
- Verify .gitignore for project-local
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**
- skills/collaboration/brainstorming (Phase 4)
- Any skill needing isolated workspace

**Pairs with:**
- skills/collaboration/finishing-a-development-branch (cleanup)
- skills/collaboration/executing-plans (work happens here)
