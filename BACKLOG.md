# Superpowers Skills Enhancement Backlog

This file tracks planned enhancements and polish for the superpowers skills library, specifically for Shopify/Ruby work.

## Completed Enhancements

### Phase 1: Essential Daily Tools ✅
1. **Enhanced `using-git-worktrees`** - Added Shopify monorepo (`dev tree`) patterns and commands
2. **Created `shopify-code-search`** - Comprehensive Grokt search syntax and workflows
3. **Enhanced `test-driven-development`** - Ruby/Minitest/Shopify patterns and coverage requirements
4. **Created `rails-database-migrations`** - Safe zero-downtime migration patterns

### Phase 2: Important Enhancements ✅
5. **Enhanced `verification-before-completion`** - Shopify-specific verification commands (`dev test`, `dev coverage`)
6. **Enhanced `defense-in-depth`** - Rails validation layers example (DB constraints → Model → Controller → Service)

---

## Phase 3: Polish (Future Work)

### Observability and Debugging

#### `shopify-observability-debugging`
**Priority:** Medium
**Type:** New Skill

Use Observe to investigate production issues before attempting fixes.

**When to use:**
- Production bugs
- Performance issues
- Understanding real user impact

**Key patterns:**
- Query error groups by service/slice
- Trace requests using trace_id
- Correlate logs across signals using request_id
- Query metrics for performance patterns
- Time range selection (default 1-4h for errors, longer for trends)

**Integration:**
- Works with Systematic Debugging - "Check production data before debugging locally"
- Uses MCP tools: `get_error_groups`, `get_error_group_by_grouping_hash`, `query_dataset`, `instant_metrics_query`, `range_metrics_query`

---

#### Enhanced `systematic-debugging`
**Priority:** Medium
**Type:** Enhancement

Add Ruby-specific debugging patterns.

**Add:**
- Phase 0: "Check Observe first for production context" (if production bug)
- Ruby debugging tools: `binding.pry` vs `debugger`, reading Ruby stack traces
- Query error groups for similar failures
- Check traces for request flow
- Rails-specific debugging patterns

---

### Ruby/Rails Performance

#### `ruby-performance-patterns`
**Priority:** Low
**Type:** New Skill (only if performance work is common)

Ruby-specific performance considerations.

**Patterns:**
- N+1 query detection (`includes`, `eager_load`, `preload`)
- Memory allocation patterns (avoid creating unnecessary objects in loops)
- String freezing for constants
- When to use `find_each` vs `find_in_batches` vs `each`
- Memoization patterns (`@var ||= expensive_calc`)
- Benchmarking techniques

**When to use:**
- Performance optimization tasks
- Code review for performance issues
- Investigating slow endpoints

---

### GraphQL (if relevant)

#### `shopify-graphql-evolution`
**Priority:** Low
**Type:** New Skill (only if you work with GraphQL regularly)

Safe GraphQL schema changes.

**Patterns:**
- Field deprecation before removal
- Backward-compatible type changes
- Testing schema changes
- Schema-first vs code-first patterns at Shopify
- Client migration strategies

**When to use:**
- Modifying GraphQL schemas
- Deprecating fields
- Adding new types/fields

---

## Research Questions

Before implementing Phase 3 skills, clarify:

### For All Skills
1. **Testing framework:** Does Shopify use Minitest or RSpec? (Answered: Minitest)
2. **Monorepo structure:** Single repo with zones or multiple repos? (Answered: Monorepo with zones)

### For Observability Skill
3. **Observe usage:** How frequently do you debug production issues using Observe?
4. **Common workflows:** What are the typical Observe query patterns?

### For Performance Skill
5. **Performance work:** Is Ruby performance optimization a regular concern?
6. **Profiling tools:** What profiling tools does Shopify use?

### For GraphQL Skill
7. **GraphQL usage:** Do you work with GraphQL schemas regularly?
8. **Schema management:** What tools/patterns does Shopify use for GraphQL?

---

## Other Potential Enhancements

### Low Priority Ideas

- **Sorbet/RBS type annotations** - If Shopify heavily uses Sorbet
- **Background job patterns** - Sidekiq/delayed_job best practices
- **Rails caching strategies** - Fragment caching, Russian doll caching
- **Security patterns** - Common Rails security pitfalls
- **API versioning** - Strategies for evolving APIs at scale

---

## Meta Skills Considerations

### Skills That May Need Review

- **`testing-skills-with-subagents`** - Very meta, potentially mark as "advanced/meta" in description
- **`sharing-skills`** - Only relevant if contributing back upstream; mark as optional

---

## Notes

- Focus on **high-frequency, high-impact** patterns first
- Avoid over-engineering - only create skills for patterns you actually use
- Review existing Shopify codebases with Grokt before creating new skills to ensure patterns are current
- Keep skills actionable - prefer concrete examples over abstract principles
