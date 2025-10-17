# Language-Specific Examples

## Structure

Each skill with language-specific examples follows this pattern:

```
skill-name/
├── SKILL.md              # Core principles + Ruby/Rails examples (default)
├── examples/
│   ├── typescript.md     # TypeScript-specific patterns
│   ├── python.md         # Python-specific patterns
│   └── go.md             # Go-specific patterns
```

## Philosophy

**Main SKILL.md:**
- Core principles (language-agnostic)
- Ruby/Rails examples throughout
- References to language-specific files

**examples/<lang>.md:**
- Focus on syntax and tooling differences
- Avoid repeating core principles
- Include one complete example in that language

## Languages

- **Ruby/Rails** - Default, Minitest, RSpec, Rails testing
- **TypeScript** - Jest, Vitest, React Testing Library
- **Python** - pytest, Django testing, unittest
- **Go** - testing package, testify, table-driven tests

## Using Language-Specific Examples

1. **Read the main SKILL.md first** - Understand the core principles and patterns
2. **Check if your language has examples** - Look in the skill's `examples/` directory
3. **Use language examples as reference** - See how the same principles apply in your language
4. **Follow the pattern** - Apply the skill's methodology using your language's idioms

## Contributing Language Examples

When adding examples for a new language:

1. Focus on **how** to apply the pattern in that language
2. Don't repeat the **why** - that's in the main SKILL.md
3. Include realistic, complete examples
4. Follow the language's testing conventions
5. See `docs/examples-checklist.md` for quality criteria
