---
name: Shopify-Code-Search
description: Use before implementing features or fixing bugs to find existing patterns, conventions, and examples across Shopify's codebase using Grokt search - prevents reinventing patterns and violating conventions
---

# Shopify Code Search (Grokt)

## Overview

Search Shopify's codebase to understand patterns, find examples, and discover conventions before implementing.

**Core principle:** Find how it's done before deciding how to do it.

**Announce at start:** "I'm using the Shopify Code Search skill to find existing patterns."

## When to Use

**Always search before:**
- Implementing a new feature (find similar implementations)
- Using an unfamiliar API or library (find usage examples)
- Writing tests for a pattern you haven't tested before (find test examples)
- Fixing a bug in unfamiliar code (understand the context)
- Deciding on naming conventions (find existing patterns)
- Understanding architectural patterns (see how it's structured)

**Red flags that mean you should search:**
- "I wonder if there's a pattern for this"
- "How do others handle X at Shopify?"
- "Is there a helper/utility for this?"
- "What's the convention for naming this?"
- "Has someone solved this already?"

## Query Syntax

### Basic Search

```ruby
# Single term
"EmailValidator"

# Multiple terms (AND)
"email validation"

# OR operator
"EmailValidator or EmailChecker"

# Phrases (exact match)
'"validates email format"'
```

### Case Sensitivity

By default, lowercase terms are case-insensitive, uppercase terms are case-sensitive:

```ruby
# Mixed case sensitivity
"class Customer"      # "class" (any case) AND "Customer" (exact case)

# Force case sensitive for all terms
"class customer case:yes"

# Exact phrase (case sensitive)
'"class Customer"'
```

### File and Language Scoping

```ruby
# Search in specific file types
"EmailValidator file:test"           # Files with "test" in name
"validates_email lang:ruby"          # Ruby files only
'f:\.rb$'                            # Files ending in .rb (regex)
"GraphQL -file:test"                 # Exclude test files

# Common languages
"authentication lang:ruby"
"useEffect lang:tsx"
"SELECT lang:sql"
```

### Repository Scoping

```ruby
# Repository name contains
"authentication r:identity"          # Repos with "identity" in name

# Repository name regex
'r:/Shopify$'                        # Repos ending with "Shopify"
"payment r:core"                     # Repos with "core" in name

# Exclude repositories
"feature -r:archived"                # Exclude archived repos
```

### Branch Scoping

```ruby
# Branch name contains
"feature b:main"                     # In main branch
"experiment b:HEAD"                  # In default branch
"refactor b:stable"                  # In branches with "stable"
```

### Excluding Results

```ruby
# Exclude terms
"customer -archived"                 # Has "customer", not "archived"

# Exclude multiple terms
"email -(spam test)"                 # Has "email", not "spam" AND not "test"

# Escape spaces in exclusions
'payment -Order\ Item'               # Exclude "Order Item" (two words)

# Exclude files
"validation -file:test"              # Exclude test files
```

### Symbol Search

Find where symbols are defined (classes, methods, constants):

```ruby
"sym:EmailValidator"                 # Find EmailValidator definition
"sym:validate_email"                 # Find validate_email method definition
"sym:ADMIN_ROLE"                     # Find ADMIN_ROLE constant
```

### Regex Search

```ruby
# Regex pattern
'email.*validation'                  # "email" followed by "validation"
'def \w+_validator'                  # Method definitions ending in _validator
'class [A-Z]\w+Service'              # Class names ending in Service
```

## Scope Selection

Choose the right scope for your search:

| Scope | Use When | Includes |
|-------|----------|----------|
| **optimal** (default) | Most searches | Active repos, excludes archived/bootcamp |
| **configs** | Looking for config patterns | Configuration files (.yml, .json, etc.) |
| **everything** | Need complete results | All repos and files |
| **private** | Shopify-internal only | Private repos only |
| **public** | Open-source patterns | Public repos only |

**Default to `optimal`** unless you have a specific reason.

## Common Search Patterns

### Finding Similar Implementations

```ruby
# Find how others implement a feature
"class EmailValidator lang:ruby"

# Find test patterns
"test 'validates email' lang:ruby"

# Find GraphQL type definitions
"type Customer lang:graphql"
```

### Finding API Usage

```ruby
# How is this gem/library used?
"Stripe.create_payment lang:ruby"

# How is this method called?
"sym:process_payment"

# Find method calls with specific patterns
'\.send_email\('
```

### Understanding Conventions

```ruby
# Naming patterns for services
"class.*Service lang:ruby r:core"

# Test naming conventions
'test "#\w+" lang:ruby file:test'

# Migration patterns
"class.*Migration lang:ruby file:migrate"
```

### Finding Helpers/Utilities

```ruby
# Is there a helper for this?
"email helper lang:ruby"
"date format utility"

# Find existing validators
"sym:*Validator lang:ruby"
```

### Debugging/Understanding Code

```ruby
# Find where this error is raised
'"Invalid email format"'

# Find all usages of a method
"validate_email_format"

# Find similar error handling
"rescue EmailValidationError"
```

## Workflow Pattern

### Before Implementing

```
1. Search for similar implementations
   Query: "class EmailValidator lang:ruby"

2. Find test patterns
   Query: "test 'validates email' lang:ruby file:test"

3. Check for existing utilities
   Query: "email validation helper"

4. Review results → Understand pattern → Implement consistently
```

### Example: Implementing Email Validation

```ruby
# Step 1: Find existing validators
Query: "class.*Validator lang:ruby r:core"
Result: Discover naming pattern: *Validator classes

# Step 2: Find validation patterns
Query: "validates :email lang:ruby"
Result: See ActiveModel validation patterns

# Step 3: Find test patterns
Query: "test 'validates.*email' lang:ruby file:test"
Result: Understand test structure expectations

# Step 4: Check for utilities
Query: "email validation helper"
Result: Discover EmailValidationHelper already exists

Decision: Use existing helper, follow established pattern
```

## Pagination and Limits

```ruby
# Default: 20 results per page
grokt_search(query: "EmailValidator", per_page: 20, page: 1)

# Get more results
grokt_search(query: "EmailValidator", per_page: 50, page: 1)

# Next page
grokt_search(query: "EmailValidator", page: 2)
```

**Start with page 1, default per_page.** Only paginate if you need more examples.

## Red Flags - Search First

**Never:**
- Implement without checking existing patterns
- Guess at naming conventions
- Assume there's no utility for common tasks
- Copy patterns from external sources without checking Shopify conventions

**Always:**
- Search before implementing
- Review multiple examples to understand the pattern
- Follow established conventions even if different from your preference
- Check test patterns before writing tests

## Integration with Other Skills

**Pairs with:**
- **Test-Driven Development**: Search for test patterns before writing tests
- **Brainstorming**: Search for similar features during design phase
- **Systematic Debugging**: Search for error messages and similar fixes

**Before using:**
- Brainstorming skill (to understand what you're building)

**After using:**
- Test-Driven Development skill (implement following discovered patterns)

## Quick Reference

| Task | Example Query |
|------|---------------|
| Find class definition | `sym:EmailValidator` |
| Find method usage | `validate_email lang:ruby` |
| Find test patterns | `test "validates" lang:ruby file:test` |
| Find in specific repo | `authentication r:identity` |
| Exclude test files | `EmailValidator -file:test` |
| Find config patterns | `email: lang:yml` (use scope: configs) |
| Case-sensitive search | `class Customer case:yes` |
| Regex pattern | `def \w+_validator` |

## Example Workflows

### Workflow 1: Adding New Validation

```
Task: Add phone number validation to User model

1. Search for existing phone validators
   grokt_search("sym:PhoneValidator lang:ruby")

2. Find validation patterns
   grokt_search("validates :phone lang:ruby")

3. Find test patterns
   grokt_search('test "validates.*phone" lang:ruby file:test')

4. Review results → Follow pattern → Implement
```

### Workflow 2: Understanding Unfamiliar Code

```
Task: Fix bug in payment processing

1. Find similar payment code
   grokt_search("payment processing lang:ruby r:billing")

2. Search for error message
   grokt_search('"Payment failed: invalid card"')

3. Find related tests
   grokt_search("payment processing file:test lang:ruby")

4. Understand context → Fix bug → Follow TDD
```

### Workflow 3: Using New API

```
Task: Integrate with new internal API

1. Find API usage examples
   grokt_search("CustomerAPI.create lang:ruby")

2. Find test patterns for API calls
   grokt_search("CustomerAPI file:test lang:ruby")

3. Check for helpers/wrappers
   grokt_search("CustomerAPI helper")

4. Use established pattern → Write tests → Implement
```

## Common Mistakes

**Implementing without searching**
- **Problem:** Reinvent patterns, violate conventions, miss existing utilities
- **Fix:** Always search first, review multiple examples

**Too narrow query**
- **Problem:** Miss relevant results
- **Fix:** Start broad, refine with scoping

**Ignoring scope parameter**
- **Problem:** Results polluted with archived/bootcamp code
- **Fix:** Use default `optimal` scope

**Not checking multiple examples**
- **Problem:** Follow one-off pattern instead of convention
- **Fix:** Review 5-10 results to identify the pattern

**Copying blindly**
- **Problem:** Copy outdated or wrong patterns
- **Fix:** Understand the pattern, adapt to your context
