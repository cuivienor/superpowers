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
