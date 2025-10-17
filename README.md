# Superpowers

Give Claude Code superpowers with a comprehensive skills library of proven techniques, patterns, and tools.

## Attribution

This project was originally created by [Jesse Vincent](https://github.com/obra) and is licensed under the MIT License. This fork has been substantially modified with new skills and enhancements for personal use.

## What You Get

- **Testing Skills** - TDD, async testing, anti-patterns
- **Debugging Skills** - Systematic debugging, root cause tracing, verification
- **Collaboration Skills** - Brainstorming, planning, code review, parallel agents
- **Meta Skills** - Creating, testing, and contributing skills

Plus:
- **Slash Commands** - `/superpowers:brainstorm`, `/superpowers:write-plan`, `/superpowers:execute-plan`

## Installation

### Via Plugin Marketplace

```bash
# In Claude Code
/plugin marketplace add cuivienor/cuivienor-marketplace
/plugin install superpowers@cuivienor-marketplace
```

### Verify Installation

```bash
# Check that commands appear
/help

# Should see:
# /superpowers:brainstorm - Interactive design refinement using Socratic method
# /superpowers:write-plan - Create detailed implementation plans
# /superpowers:execute-plan - Execute plan in batches with review checkpoints
```

## Quick Start

### Using Skills

Skills are available through the Skill tool in Claude Code. Claude will automatically detect and use relevant skills based on the task at hand.

### Using Slash Commands

**Brainstorm a design:**
```
/superpowers:brainstorm
```

**Create an implementation plan:**
```
/superpowers:write-plan
```

**Execute a plan:**
```
/superpowers:execute-plan
```

## What's Inside

### Skills Library

**Testing** (`skills/testing/`)
- test-driven-development - RED-GREEN-REFACTOR cycle
- condition-based-waiting - Async test patterns
- testing-anti-patterns - Common pitfalls to avoid

**Debugging** (`skills/debugging/`)
- systematic-debugging - 4-phase root cause process
- root-cause-tracing - Find the real problem
- verification-before-completion - Ensure it's actually fixed
- defense-in-depth - Multiple validation layers

**Collaboration** (`skills/collaboration/`)
- brainstorming - Socratic design refinement
- writing-plans - Detailed implementation plans
- executing-plans - Batch execution with checkpoints
- dispatching-parallel-agents - Concurrent subagent workflows
- using-git-worktrees - Parallel development branches
- requesting-code-review - Pre-review checklist
- receiving-code-review - Responding to feedback
- finishing-a-development-branch - Completion workflows

**Shopify/Rails** (`skills/shopify/`)
- shopify-code-search - Search Shopify codebase with Grokt
- rails-database-migrations - Safe zero-downtime migrations

**Meta** (`skills/meta/`)
- writing-skills - TDD for documentation, create new skills
- testing-skills-with-subagents - Validate skill quality

### Slash Commands

Available via `/superpowers:` namespace:
- **brainstorm** - Interactive design refinement using Socratic method
- **write-plan** - Create detailed implementation plans
- **execute-plan** - Execute plans in batches with review checkpoints


## How It Works

1. **SessionStart Hook** - Injects skills context into Claude Code sessions
2. **Skills Discovery** - Claude automatically identifies relevant skills for tasks
3. **Mandatory Workflow** - Skills become required when they exist for your task

## Philosophy

- **Test-Driven Development** - Write tests first, always
- **Systematic over ad-hoc** - Process over guessing
- **Complexity reduction** - Simplicity as primary goal
- **Evidence over claims** - Verify before declaring success
- **Domain over implementation** - Work at problem level, not solution level

## License

MIT License - see LICENSE file for details

## Support

- **Issues**: https://github.com/cuivienor/superpowers/issues
- **Repository**: https://github.com/cuivienor/superpowers
