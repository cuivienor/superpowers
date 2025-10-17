# Examples Quality Checklist

Before submitting language-specific examples, ensure they meet these quality criteria:

## Content Quality

- [ ] **Core principles stay in main SKILL.md** - Don't duplicate the "why", only show the "how"
- [ ] **Language file focuses on differences, not principles** - Syntax, tooling, and idioms specific to that language
- [ ] **At least one complete example in target language** - Full RED-GREEN-REFACTOR cycle or complete workflow
- [ ] **Code is syntactically correct** - All code examples are valid and runnable
- [ ] **Examples are realistic and practical** - Based on real-world usage patterns, not contrived examples

## Structure & Organization

- [ ] **Language reference section at top of SKILL.md** - Lists all available language examples with descriptive text
- [ ] **Consistent naming: `examples/<lang>.md`** - Use lowercase language name (typescript, python, go, ruby)
- [ ] **Each language file has proper header** - Include note pointing back to main SKILL.md
- [ ] **Links use `@examples/<lang>.md` syntax** - Consistent reference format across all skills
- [ ] **Clear section headings** - Organize by concept (Test Structure, Assertions, Running Tests, etc.)

## Coverage & Completeness

### Must Include:

- [ ] **Test structure and naming** - How to organize tests in this language
- [ ] **Assertion patterns** - Language-specific assertion syntax and best practices
- [ ] **Mocking/stubbing approaches** - If applicable, how to handle test doubles
- [ ] **Running tests** - Commands and flags for running tests
- [ ] **Coverage measurement** - How to check test coverage in this language

### Should Include (when applicable):

- [ ] **Framework-specific features** - Unique capabilities of the testing framework
- [ ] **Common pitfalls** - Language-specific gotchas and anti-patterns
- [ ] **Tool integration** - IDE support, CI/CD considerations
- [ ] **Performance considerations** - If relevant to the skill

## Quality & Accuracy

- [ ] **Examples tested in real project** - Verify examples actually work, not just theoretical
- [ ] **No language-specific opinions in language-agnostic sections** - Keep main SKILL.md neutral
- [ ] **Clear, concise explanations** - Just enough context, not verbose
- [ ] **Follows existing skill tone and style** - Match the voice of other skills
- [ ] **Code examples use good/bad comparisons** - Show both correct and incorrect patterns where helpful
- [ ] **Consistent formatting** - Follow markdown conventions, use syntax highlighting

## Cross-References

- [ ] **Main SKILL.md references all language examples** - Complete list in Language-Specific Examples section
- [ ] **Language examples reference main SKILL.md** - Clear note at the top of each file
- [ ] **No broken links** - All `@examples/` references point to existing files
- [ ] **Consistent terminology** - Same terms used across all language examples

## Documentation

- [ ] **Each example has explanatory text** - Don't just dump code, explain what's happening
- [ ] **Comments in code when helpful** - Clarify non-obvious parts
- [ ] **Good/Bad examples clearly marked** - Use consistent markers (✅ GOOD, ❌ BAD)
- [ ] **"Why" explanations for anti-patterns** - Explain what makes the bad example bad

## Validation Steps

Before submitting:

1. **Read through the main SKILL.md** - Ensure you understand the core principles
2. **Check for duplication** - Make sure you're not repeating what's already in SKILL.md
3. **Test the code** - Run all examples in a real project to verify they work
4. **Check all links** - Verify `@examples/` references resolve correctly
5. **Review consistency** - Compare with other language examples in the same skill
6. **Get feedback** - Have someone unfamiliar with the language review for clarity

## Common Mistakes to Avoid

- ❌ Repeating the core principles from SKILL.md in the language file
- ❌ Only showing syntax without complete, runnable examples
- ❌ Using contrived examples that don't reflect real-world usage
- ❌ Forgetting to show how to run tests and check coverage
- ❌ Not testing examples before submitting
- ❌ Mixing multiple language versions in one file
- ❌ Breaking the established structure pattern
- ❌ Adding personal opinions about language superiority

## File Template

Use this structure for new language examples:

```markdown
# [Skill Name]: [Language] Examples

> **Note:** This file contains [Language]-specific patterns. For core [skill] principles, see the main SKILL.md.

## Language-Specific Patterns

### Test Structure
[How to organize tests in this language]

### Assertions
[Language-specific assertion syntax]

### [Other relevant sections based on skill type]

### Running Tests
[Commands and tools]

### Coverage Requirements
[How to measure coverage]

### Complete Example: [Real-world scenario]
[Full example showing the skill in action]

### [Language]-Specific Features
[Unique capabilities or considerations]
```

## Success Criteria

A high-quality language example should:

1. **Enable immediate usage** - Developer can copy patterns directly into their project
2. **Complement, not duplicate** - Works with main SKILL.md, doesn't replace it
3. **Reflect real usage** - Based on actual projects and conventions
4. **Be maintainable** - Easy to update as language/frameworks evolve
5. **Add clear value** - Demonstrates something that wasn't obvious from main SKILL.md
