# Language-Specific Examples Implementation Plan

> **For Claude:** Use `skills/executing-plans/SKILL.md` to implement this plan task-by-task.

**Goal:** Refactor skills to use Ruby/Rails as default with TypeScript, Python, and Go examples in separate files.

**Architecture:** Keep core principles language-agnostic in main SKILL.md files. Extract language-specific patterns to `examples/<lang>.md` files. Start with test-driven-development as the pilot, then apply pattern to other skills.

**Tech Stack:** Markdown, Ruby/Rails/Minitest (default), TypeScript/Jest/Vitest, Python/pytest, Go/testing

---

## Task 1: Refactor test-driven-development Skill

**Goal:** Transform TDD skill to use Ruby/Rails as default with language-specific examples extracted.

**Files:**
- Modify: `skills/test-driven-development/SKILL.md`
- Create: `skills/test-driven-development/examples/typescript.md`
- Create: `skills/test-driven-development/examples/python.md`
- Create: `skills/test-driven-development/examples/go.md`

### Step 1: Create examples directory

```bash
mkdir -p skills/test-driven-development/examples
```

### Step 2: Add language reference section to SKILL.md

**Location:** After frontmatter, before "## Overview"

Add:
```markdown
## Language-Specific Examples

This skill uses **Ruby/Rails and Minitest** as the default.

For language-specific patterns, see:
- **TypeScript** (Jest, Vitest): `@examples/typescript.md`
- **Python** (pytest): `@examples/python.md`
- **Go** (testing): `@examples/go.md`

---
```

### Step 3: Move TypeScript examples to examples/typescript.md

**Extract from SKILL.md lines 76-164:**
- The TypeScript "RED - Write Failing Test" example (lines 76-106)
- The TypeScript "GREEN - Minimal Code" example (lines 134-164)
- The TypeScript Bug Fix example (lines 290-326)

**Create:** `skills/test-driven-development/examples/typescript.md`

**Content structure:**
```markdown
# Test-Driven Development: TypeScript Examples

> **Note:** This file contains TypeScript-specific patterns. For core TDD principles, see the main SKILL.md.

## Language-Specific Patterns

### Test Structure
[How TypeScript/Jest/Vitest tests are structured]

### RED - Write Failing Test
[TypeScript example from main file]

### GREEN - Minimal Code
[TypeScript example from main file]

### Running Tests
[TypeScript-specific commands]

### Complete Example: Bug Fix
[TypeScript bug fix example]
```

### Step 4: Replace TypeScript examples with Ruby in SKILL.md

**Replace TypeScript example at lines 76-106 with Ruby:**

```ruby
test "#retry_operation retries failed operations 3 times" do
  attempts = 0
  operation = lambda do
    attempts += 1
    raise StandardError, "fail" if attempts < 3
    "success"
  end

  result = retry_operation(operation)

  assert_equal("success", result)
  assert_equal(3, attempts)
end
```

Good: Clear name, tests real behavior, one thing

```ruby
test "retry works" do
  mock = mock()
  mock.expects(:call)
    .raises(StandardError.new)
    .twice
  mock.expects(:call)
    .returns("success")

  retry_operation(mock)
end
```

Bad: Vague name, tests mock not code

**Replace TypeScript GREEN example at lines 134-164 with Ruby:**

```ruby
def retry_operation(&block)
  (0..2).each do |i|
    begin
      return block.call
    rescue StandardError => e
      raise e if i == 2
    end
  end
  raise "unreachable"
end
```

Good: Just enough to pass

```ruby
def retry_operation(
  &block,
  max_retries: 3,
  backoff: :linear,
  on_retry: nil
)
  # YAGNI
end
```

Bad: Over-engineered

### Step 5: Keep Ruby/Minitest section (lines 327-482) as-is

The section starting at line 327 "## Ruby/Minitest at Shopify" should remain unchanged - this becomes the default examples throughout the document.

### Step 6: Update Bug Fix example (lines 290-326) to use Ruby

**Replace TypeScript bug fix example with Ruby:**

```markdown
## Example: Bug Fix

**Bug:** Empty email accepted

**RED**
```ruby
test "#submit_form rejects empty email" do
  result = submit_form(email: "")

  assert_equal("Email required", result[:error])
end
```

**Verify RED**
```bash
$ /opt/dev/bin/dev test test/forms/submit_form_test.rb
FAIL: expected "Email required", got nil
```

**GREEN**
```ruby
def submit_form(data)
  if data[:email].to_s.strip.empty?
    return { error: "Email required" }
  end
  # ...
end
```

**Verify GREEN**
```bash
$ /opt/dev/bin/dev test test/forms/submit_form_test.rb
1 tests, 1 assertions, 0 failures
```

**REFACTOR**
Extract validation for multiple fields if needed.
```

### Step 7: Create examples/python.md

**Create:** `skills/test-driven-development/examples/python.md`

**Content:**
```markdown
# Test-Driven Development: Python Examples

> **Note:** This file contains Python-specific patterns. For core TDD principles, see the main SKILL.md.

## Language-Specific Patterns

### Test Structure

```python
# Test files must match test_*.py or *_test.py pattern
# tests/test_email_validator.py

import pytest
from app.validators import EmailValidator

class TestEmailValidator:
    def setup_method(self):
        """Runs before each test method"""
        self.validator = EmailValidator()

    def test_valid_returns_true_for_valid_email(self):
        assert self.validator.valid("user@example.com")

    def test_valid_returns_false_for_empty_email(self):
        assert not self.validator.valid("")

    @staticmethod
    def test_normalize_removes_whitespace():
        assert EmailValidator.normalize("  user@example.com  ") == "user@example.com"
```

**Key patterns:**
- Instance methods: `test_<method>_<behavior>`
- Static methods: `@staticmethod` decorator
- Setup: `setup_method` and `teardown_method`
- Fixtures: Use `@pytest.fixture` for shared state

### Assertions

```python
# pytest assertions use plain assert with helpful error messages
assert actual == expected
assert len(collection) == 0
assert isinstance(obj, MyClass)
assert obj.is_ready()
assert actual != expected
assert obj is not None

# pytest provides special assertions
pytest.raises(ValueError, func)
pytest.warns(UserWarning, func)
pytest.approx(3.14, 0.01)
```

### Mocking with unittest.mock

```python
from unittest.mock import Mock, patch

def test_service_calls_repository():
    # Create mock
    mock_repo = Mock()
    mock_repo.save.return_value = True

    service = UserService(mock_repo)
    result = service.create_user("test@example.com")

    # Verify call
    mock_repo.save.assert_called_once()
    assert result is True

# Patching
@patch('app.services.external_api')
def test_service_with_patch(mock_api):
    mock_api.fetch.return_value = {"status": "ok"}

    result = MyService().process()

    assert result == {"status": "ok"}
```

### Running Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_validator.py

# Run specific test
pytest tests/test_validator.py::TestEmailValidator::test_valid_returns_true_for_valid_email

# Run with verbose output
pytest -v

# Run with coverage
pytest --cov=app --cov-report=html

# Run tests matching pattern
pytest -k "email"
```

### Coverage Requirements

```bash
# Check coverage
pytest --cov=app --cov-report=term-missing

# Coverage must be ≥95% branch
pytest --cov=app --cov-branch --cov-report=term-missing

# Generate HTML report
pytest --cov=app --cov-report=html
# Open htmlcov/index.html
```

### Complete Example: Bug Fix

**Bug:** Empty email accepted by validator

**RED**
```python
def test_valid_rejects_empty_email():
    validator = EmailValidator()

    assert not validator.valid("")
    assert validator.error == "Email required"
```

**Verify RED**
```bash
$ pytest tests/test_email_validator.py::test_valid_rejects_empty_email -v
FAILED: AssertionError: assert True
```

**GREEN**
```python
class EmailValidator:
    def __init__(self):
        self.error = None

    def valid(self, email):
        if not email or not email.strip():
            self.error = "Email required"
            return False
        return True
```

**Verify GREEN**
```bash
$ pytest tests/test_email_validator.py::test_valid_rejects_empty_email -v
PASSED
```

**REFACTOR**
Extract empty check if used in multiple validators.

### Pytest-Specific Features

**Parametrized tests** (table-driven):
```python
@pytest.mark.parametrize("email,expected", [
    ("user@example.com", True),
    ("", False),
    ("   ", False),
    ("invalid", False),
])
def test_valid_with_various_inputs(email, expected):
    validator = EmailValidator()
    assert validator.valid(email) == expected
```

**Fixtures** (shared setup):
```python
@pytest.fixture
def validator():
    return EmailValidator()

def test_valid_email(validator):
    assert validator.valid("user@example.com")

def test_invalid_email(validator):
    assert not validator.valid("")
```

**Markers** (categorize tests):
```python
@pytest.mark.slow
def test_expensive_operation():
    # Run with: pytest -m slow
    pass

@pytest.mark.integration
def test_database_integration():
    # Run with: pytest -m integration
    pass
```
```

### Step 8: Create examples/go.md

**Create:** `skills/test-driven-development/examples/go.md`

**Content:**
```markdown
# Test-Driven Development: Go Examples

> **Note:** This file contains Go-specific patterns. For core TDD principles, see the main SKILL.md.

## Language-Specific Patterns

### Test Structure

```go
// Test files must end with _test.go
// email_validator_test.go

package validators

import (
    "testing"
)

func TestEmailValidator_Valid_ValidEmail(t *testing.T) {
    validator := NewEmailValidator()

    if !validator.Valid("user@example.com") {
        t.Error("expected valid email to return true")
    }
}

func TestEmailValidator_Valid_EmptyEmail(t *testing.T) {
    validator := NewEmailValidator()

    if validator.Valid("") {
        t.Error("expected empty email to return false")
    }
}

func TestEmailValidator_Normalize(t *testing.T) {
    result := EmailValidator.Normalize("  user@example.com  ")
    expected := "user@example.com"

    if result != expected {
        t.Errorf("expected %q, got %q", expected, result)
    }
}
```

**Key patterns:**
- Function naming: `Test<Type>_<Method>_<Scenario>`
- Test functions take `*testing.T` parameter
- Use `t.Error`, `t.Errorf`, `t.Fatal`, `t.Fatalf` for assertions
- No setup/teardown - use subtests or helper functions

### Assertions

Go's standard library has minimal assertions. Use conditional checks:

```go
// Basic assertions
if actual != expected {
    t.Errorf("expected %v, got %v", expected, actual)
}

if len(items) != 0 {
    t.Errorf("expected empty slice, got %d items", len(items))
}

if err != nil {
    t.Fatalf("unexpected error: %v", err)
}

if obj == nil {
    t.Fatal("expected non-nil object")
}
```

**Using testify/assert** (common third-party library):
```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestExample(t *testing.T) {
    assert.Equal(t, expected, actual)
    assert.Empty(t, collection)
    assert.NotNil(t, obj)
    assert.True(t, condition)
    assert.Error(t, err)
    assert.NoError(t, err)
}
```

### "Mocking" with Interfaces

Go doesn't have traditional mocking. Instead, use interfaces and dependency injection:

```go
// Define interface for dependency
type Repository interface {
    Save(user User) error
}

// Production implementation
type PostgresRepository struct {
    db *sql.DB
}

func (r *PostgresRepository) Save(user User) error {
    // Real database logic
}

// Test implementation
type MockRepository struct {
    SaveCalled bool
    SaveError  error
}

func (m *MockRepository) Save(user User) error {
    m.SaveCalled = true
    return m.SaveError
}

// Test uses mock
func TestUserService_Create(t *testing.T) {
    mockRepo := &MockRepository{}
    service := NewUserService(mockRepo)

    err := service.Create(User{Email: "test@example.com"})

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if !mockRepo.SaveCalled {
        t.Error("expected Save to be called")
    }
}
```

### Table-Driven Tests

Go's idiomatic pattern for testing multiple cases:

```go
func TestEmailValidator_Valid(t *testing.T) {
    tests := []struct {
        name     string
        email    string
        expected bool
    }{
        {"valid email", "user@example.com", true},
        {"empty email", "", false},
        {"whitespace only", "   ", false},
        {"missing @", "userexample.com", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            validator := NewEmailValidator()

            result := validator.Valid(tt.email)

            if result != tt.expected {
                t.Errorf("Valid(%q) = %v, expected %v",
                    tt.email, result, tt.expected)
            }
        })
    }
}
```

### Running Tests

```bash
# Run all tests in package
go test

# Run all tests in package with verbose output
go test -v

# Run specific test
go test -run TestEmailValidator_Valid

# Run tests matching pattern
go test -run Valid

# Run with coverage
go test -cover

# Generate coverage report
go test -coverprofile=coverage.out
go tool cover -html=coverage.out

# Run tests in all subdirectories
go test ./...
```

### Complete Example: Bug Fix

**Bug:** Empty email accepted by validator

**RED**
```go
func TestEmailValidator_Valid_EmptyEmail(t *testing.T) {
    validator := NewEmailValidator()

    if validator.Valid("") {
        t.Error("expected empty email to return false")
    }

    if validator.Error() != "Email required" {
        t.Errorf("expected error %q, got %q",
            "Email required", validator.Error())
    }
}
```

**Verify RED**
```bash
$ go test -run TestEmailValidator_Valid_EmptyEmail -v
--- FAIL: TestEmailValidator_Valid_EmptyEmail (0.00s)
    email_validator_test.go:X: expected empty email to return false
FAIL
```

**GREEN**
```go
type EmailValidator struct {
    err string
}

func NewEmailValidator() *EmailValidator {
    return &EmailValidator{}
}

func (v *EmailValidator) Valid(email string) bool {
    email = strings.TrimSpace(email)
    if email == "" {
        v.err = "Email required"
        return false
    }
    return true
}

func (v *EmailValidator) Error() string {
    return v.err
}
```

**Verify GREEN**
```bash
$ go test -run TestEmailValidator_Valid_EmptyEmail -v
--- PASS: TestEmailValidator_Valid_EmptyEmail (0.00s)
PASS
```

**REFACTOR**
Extract empty check if used in multiple validators.

### Go-Specific Best Practices

**Use subtests for organization:**
```go
func TestEmailValidator(t *testing.T) {
    t.Run("Valid", func(t *testing.T) {
        // Valid email tests
    })

    t.Run("Invalid", func(t *testing.T) {
        // Invalid email tests
    })
}
```

**Helper functions:**
```go
// Helper functions should call t.Helper() to get correct line numbers
func assertValid(t *testing.T, validator *EmailValidator, email string) {
    t.Helper()
    if !validator.Valid(email) {
        t.Errorf("expected %q to be valid", email)
    }
}
```

**Parallel tests:**
```go
func TestExpensiveOperation(t *testing.T) {
    t.Parallel() // Runs in parallel with other parallel tests

    // Test logic
}
```
```

### Step 9: Verify all links and references work

**Check:**
- `@examples/typescript.md` references resolve correctly
- `@examples/python.md` references resolve correctly
- `@examples/go.md` references resolve correctly
- Line numbers in examples match actual content
- Code examples are syntactically correct

### Step 10: Test the refactored skill

**Manual verification:**
1. Read `SKILL.md` and verify Ruby is the default
2. Read each `examples/*.md` file and verify completeness
3. Verify no duplication of core principles
4. Verify examples focus on language-specific differences

### Step 11: Commit changes

```bash
git add skills/test-driven-development/
git commit -m "refactor(tdd): extract language-specific examples

- Use Ruby/Rails as default throughout SKILL.md
- Extract TypeScript examples to examples/typescript.md
- Add Python/pytest examples to examples/python.md
- Add Go/testing examples to examples/go.md
- Add language reference section at top of SKILL.md

Each language file focuses on syntax and tooling differences while
core TDD principles remain in main SKILL.md."
```

---

## Task 2: Apply Pattern to testing-anti-patterns

**Goal:** Refactor testing-anti-patterns to use Ruby/Rails as default with language-specific examples.

**Files:**
- Modify: `skills/testing-anti-patterns/SKILL.md`
- Create: `skills/testing-anti-patterns/examples/typescript.md`
- Create: `skills/testing-anti-patterns/examples/python.md`
- Create: `skills/testing-anti-patterns/examples/go.md`

### Step 1: Create examples directory

```bash
mkdir -p skills/testing-anti-patterns/examples
```

### Step 2: Add language reference section

Add after frontmatter in `skills/testing-anti-patterns/SKILL.md`:

```markdown
## Language-Specific Examples

This skill uses **Ruby/Rails and Minitest** as the default.

For language-specific anti-patterns, see:
- **TypeScript** (Jest, Vitest): `@examples/typescript.md`
- **Python** (pytest): `@examples/python.md`
- **Go** (testing): `@examples/go.md`

---
```

### Step 3: Convert TypeScript examples to Ruby

Replace TypeScript examples with Ruby equivalents:

**Anti-Pattern 1** becomes:
```ruby
# ❌ BAD: Testing that the mock exists
test "renders sidebar" do
  get :show
  assert_select "[data-testid='sidebar-mock']"
end
```

**Anti-Pattern 2** becomes:
```ruby
# ❌ BAD: destroy() only used in tests
class Session
  def destroy
    @workspace_manager&.destroy_workspace(id)
    # ... cleanup
  end
end

# In tests
teardown { @session.destroy }
```

### Step 4: Create TypeScript examples file

Extract TypeScript anti-patterns to `examples/typescript.md` with:
- TS-specific mocking anti-patterns (Jest mocks)
- React Testing Library anti-patterns
- Async test anti-patterns

### Step 5: Create Python examples file

Create `examples/python.md` with Python/pytest specific anti-patterns:
- pytest fixture abuse
- Mock vs MagicMock misuse
- Patching anti-patterns

### Step 6: Create Go examples file

Create `examples/go.md` with Go-specific anti-patterns:
- Interface pollution for testing
- Over-complex test doubles
- Subtest abuse

### Step 7: Commit changes

```bash
git add skills/testing-anti-patterns/
git commit -m "refactor(testing-anti-patterns): extract language-specific examples

- Use Ruby/Rails as default throughout SKILL.md
- Extract TypeScript examples to examples/typescript.md
- Add Python-specific anti-patterns to examples/python.md
- Add Go-specific anti-patterns to examples/go.md"
```

---

## Task 3: Apply Pattern to condition-based-waiting

**Goal:** Refactor condition-based-waiting to use Ruby/Rails as default with language-specific examples.

**Files:**
- Modify: `skills/condition-based-waiting/SKILL.md`
- Rename: `skills/condition-based-waiting/example.ts` → `skills/condition-based-waiting/examples/typescript.md`
- Create: `skills/condition-based-waiting/examples/ruby.md`
- Create: `skills/condition-based-waiting/examples/python.md`
- Create: `skills/condition-based-waiting/examples/go.md`

### Step 1: Create examples directory and move TypeScript

```bash
mkdir -p skills/condition-based-waiting/examples
mv skills/condition-based-waiting/example.ts skills/condition-based-waiting/examples/typescript.md
```

### Step 2: Convert example.ts to markdown

Wrap the TypeScript code in markdown with explanatory text in `examples/typescript.md`.

### Step 3: Add language reference section to SKILL.md

### Step 4: Replace core pattern example with Ruby

Replace TypeScript example with Ruby equivalent showing condition-based waiting.

### Step 5: Create Ruby examples file

Create `examples/ruby.md` with Rails-specific patterns:
- Capybara wait helpers
- RSpec eventually matchers
- Minitest wait patterns

### Step 6: Create Python examples file

Create `examples/python.md` with pytest patterns:
- pytest-timeout
- Custom wait helpers
- asyncio wait patterns

### Step 7: Create Go examples file

Create `examples/go.md` with Go patterns:
- Channel-based waiting
- Context with timeout
- Testing async operations

### Step 8: Commit changes

```bash
git add skills/condition-based-waiting/
git commit -m "refactor(condition-based-waiting): extract language-specific examples

- Use Ruby/Rails as default throughout SKILL.md
- Move TypeScript example to examples/typescript.md
- Add Ruby/Rails examples to examples/ruby.md
- Add Python examples to examples/python.md
- Add Go examples to examples/go.md"
```

---

## Task 4: Apply Pattern to defense-in-depth

**Goal:** Refactor defense-in-depth to use Ruby/Rails as default with language-specific examples.

**Files:**
- Modify: `skills/defense-in-depth/SKILL.md`
- Create: `skills/defense-in-depth/examples/typescript.md`
- Create: `skills/defense-in-depth/examples/python.md`
- Create: `skills/defense-in-depth/examples/go.md`

### Step 1: Create examples directory

```bash
mkdir -p skills/defense-in-depth/examples
```

### Step 2: Add language reference section

### Step 3: Keep Ruby examples as primary

The skill already has good Rails examples (email validation with DB constraints, model validations, etc.). Keep these as the main examples.

### Step 4: Extract TypeScript portions

Extract TypeScript validation examples to `examples/typescript.md`.

### Step 5: Create Python examples file

Create `examples/python.md` with Django validation layers:
- Database constraints
- Model validation
- Form validation
- View validation

### Step 6: Create Go examples file

Create `examples/go.md` with Go validation patterns:
- Database constraints (with sqlc or similar)
- Struct validation
- API handler validation
- Middleware validation

### Step 7: Commit changes

```bash
git add skills/defense-in-depth/
git commit -m "refactor(defense-in-depth): extract language-specific examples

- Keep Ruby/Rails as default with comprehensive example
- Extract TypeScript examples to examples/typescript.md
- Add Django examples to examples/python.md
- Add Go examples to examples/go.md"
```

---

## Task 5: Documentation and Verification

### Step 1: Update README.md

Update `README.md` to mention language-specific examples:

```markdown
## Language Support

Skills use **Ruby/Rails** as the default language throughout examples.

**Language-specific patterns** are available for:
- TypeScript (Jest, Vitest, React Testing Library)
- Python (pytest, Django)
- Go (testing package, testify)

Skills with language-specific examples:
- test-driven-development
- testing-anti-patterns
- condition-based-waiting
- defense-in-depth

When using these skills in a non-Ruby project, check the `examples/` directory for language-specific patterns.
```

### Step 2: Create examples/README.md template

Create a template explaining the examples structure:

```markdown
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
```

### Step 3: Test all examples

Verify each example file:
- [ ] Syntax is correct
- [ ] Examples are complete
- [ ] Links work
- [ ] No duplication of core principles
- [ ] Each file stands alone

### Step 4: Create verification checklist

Create `docs/examples-checklist.md`:

```markdown
# Examples Quality Checklist

Before submitting language-specific examples:

## Content
- [ ] Core principles stay in main SKILL.md
- [ ] Language file focuses on differences, not principles
- [ ] At least one complete example in target language
- [ ] Code is syntactically correct
- [ ] Examples are realistic and practical

## Structure
- [ ] Language reference section at top of SKILL.md
- [ ] Consistent naming: `examples/<lang>.md`
- [ ] Each language file has proper frontmatter
- [ ] Links use `@examples/<lang>.md` syntax

## Coverage
- [ ] Test structure and naming
- [ ] Assertion patterns
- [ ] Mocking/stubbing approaches
- [ ] Running tests
- [ ] Coverage measurement

## Quality
- [ ] Examples tested in real project
- [ ] No language-specific opinions in language-agnostic sections
- [ ] Clear, concise explanations
- [ ] Follows existing skill tone and style
```

### Step 5: Commit documentation

```bash
git add README.md docs/examples-checklist.md
git commit -m "docs: add language-specific examples documentation

- Update README with language support section
- Add examples quality checklist
- Document examples structure and philosophy"
```

---

## Execution Notes

**Total estimated time:** 6-8 hours

**Order of execution:**
1. Task 1 (TDD) - Most complex, validates pattern
2. Task 2 (Anti-patterns) - Similar to TDD, natural companion
3. Task 3 (Condition-based waiting) - Different skill type (async)
4. Task 4 (Defense-in-depth) - Validation patterns
5. Task 5 (Documentation) - Final polish

**Verification between tasks:**
- Review each completed skill before moving to next
- Ensure consistency in structure and style
- Test that examples are complete and accurate

**Future work:**
- Add more languages (JavaScript, Java, Rust)
- Apply pattern to other skills (root-cause-tracing, systematic-debugging)
- Consider adding .skill-context.json for automatic loading
