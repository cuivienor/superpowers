# Testing Anti-Patterns: Go Examples

> **Note:** This file contains Go-specific anti-patterns. For core testing principles, see the main SKILL.md.

## Language-Specific Anti-Patterns

### Anti-Pattern 1: Testing Mock Behavior

**The violation:**
```go
// ❌ BAD: Testing that the mock exists
func TestRendersSidebar(t *testing.T) {
    page := NewPage()
    html := page.Render()

    if !strings.Contains(html, "sidebar-mock") {
        t.Error("expected sidebar-mock to be present")
    }
}
```

**Why this is wrong:**
- You're verifying the mock works, not that the page works
- Test passes when mock is present, fails when it's not
- Tells you nothing about real behavior

**The fix:**
```go
// ✅ GOOD: Test real component or don't mock it
func TestRendersSidebar(t *testing.T) {
    page := NewPage()
    html := page.Render()

    if !strings.Contains(html, "<nav") {
        t.Error("expected navigation element")
    }
}
```

### Anti-Pattern 2: Test-Only Methods in Production Structs

**The violation:**
```go
// ❌ BAD: Destroy() only used in tests
type Session struct {
    workspaceManager *WorkspaceManager
    id               string
}

func (s *Session) Destroy() {  // Looks like production API!
    if s.workspaceManager != nil {
        s.workspaceManager.DestroyWorkspace(s.id)
    }
    // ... cleanup
}

// In tests
func TestSomething(t *testing.T) {
    session := NewSession()
    defer session.Destroy()
    // ... test
}
```

**Why this is wrong:**
- Production struct polluted with test-only code
- Dangerous if accidentally called in production
- Violates YAGNI and separation of concerns

**The fix:**
```go
// ✅ GOOD: Test utilities handle test cleanup
// Session has no Destroy() - it's stateless in production

// In testing/testutil/session.go
func CleanupSession(session *Session) {
    workspace := session.WorkspaceInfo()
    if workspace != nil {
        workspaceManager.DestroyWorkspace(workspace.ID)
    }
}

// In tests
func TestSomething(t *testing.T) {
    session := NewSession()
    defer testutil.CleanupSession(session)
    // ... test
}
```

### Anti-Pattern 3: Mocking Without Understanding

**The violation:**
```go
// ❌ BAD: Mock breaks test logic
func TestDetectsDuplicateServer(t *testing.T) {
    // Mock prevents config write that test depends on!
    mockCatalog := &MockToolCatalog{
        DiscoverAndCacheToolsFunc: func() error { return nil },
    }

    addServer(config, mockCatalog)
    err := addServer(config, mockCatalog)  // Should error - but won't!

    if err == nil {
        t.Error("expected duplicate server error")
    }
}
```

**Why this is wrong:**
- Mocked method had side effect test depended on (writing config)
- Over-mocking to "be safe" breaks actual behavior
- Test passes for wrong reason or fails mysteriously

**The fix:**
```go
// ✅ GOOD: Mock at correct level
func TestDetectsDuplicateServer(t *testing.T) {
    // Mock the slow part, preserve behavior test needs
    mockServerManager := &MockServerManager{
        StartServerFunc: func() error { return nil },
    }

    addServer(config, mockServerManager)  // Config written
    err := addServer(config, mockServerManager)  // Duplicate detected ✓

    if err == nil {
        t.Error("expected duplicate server error")
    }
}
```

### Anti-Pattern 4: Incomplete Mocks

**The violation:**
```go
// ❌ BAD: Partial mock - only fields you think you need
type MockResponse struct {
    Status string
    Data   map[string]interface{}
    // Missing: Metadata that downstream code uses
}

// Later: panics when code accesses response.Metadata.RequestID
```

**The fix:**
```go
// ✅ GOOD: Mirror real API completeness
type MockResponse struct {
    Status   string
    Data     map[string]interface{}
    Metadata ResponseMetadata  // All fields real API returns
}

type ResponseMetadata struct {
    RequestID string
    Timestamp int64
}
```

## Go-Specific Anti-Patterns

### Anti-Pattern: Interface Pollution for Testing

**The violation:**
```go
// ❌ BAD: Creating interfaces only for mocking
// production.go
type UserRepository interface {
    Save(user User) error
    FindByID(id string) (*User, error)
    FindByEmail(email string) (*User, error)
    Update(user User) error
    Delete(id string) error
    List(limit, offset int) ([]User, error)
    Count() (int, error)
}

// Only one production implementation
type PostgresUserRepository struct {
    db *sql.DB
}

// Interface created solely so tests can mock it
```

**Why this is wrong:**
- Interfaces should be defined by consumers, not implementations
- "Accept interfaces, return structs" principle violated
- Over-abstraction for testing convenience

**The fix:**
```go
// ✅ GOOD: Define minimal interfaces where needed
// service.go (consumer defines interface)
type UserSaver interface {
    Save(user User) error  // Only what service needs
}

type UserService struct {
    saver UserSaver  // Accepts minimal interface
}

// production.go
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) Save(user User) error {
    // Implementation
}

// test_helper.go
type FakeUserRepository struct {
    users []User
}

func (r *FakeUserRepository) Save(user User) error {
    r.users = append(r.users, user)
    return nil
}

// Both satisfy UserSaver without explicit interface definition
```

### Anti-Pattern: Overusing testify/mock

**The violation:**
```go
// ❌ BAD: Complex mock setup with testify/mock
func TestUserService(t *testing.T) {
    mockRepo := new(MockUserRepository)
    mockRepo.On("FindByID", "123").Return(&User{ID: "123"}, nil)
    mockRepo.On("Update", mock.Anything).Return(nil)
    mockRepo.On("Save", mock.MatchedBy(func(u User) bool {
        return u.Email != ""
    })).Return(nil)

    service := NewUserService(mockRepo)
    // Complex test setup continues...

    mockRepo.AssertExpectations(t)
}
```

**Why this is wrong:**
- Mock setup is more complex than implementation
- Fragile - breaks when call order changes
- Hard to understand what's being tested

**The fix:**
```go
// ✅ GOOD: Simple test double
type FakeUserRepository struct {
    users      map[string]*User
    saveCalled bool
    saveError  error
}

func NewFakeUserRepository() *FakeUserRepository {
    return &FakeUserRepository{
        users: make(map[string]*User),
    }
}

func (r *FakeUserRepository) Save(user User) error {
    r.saveCalled = true
    if r.saveError != nil {
        return r.saveError
    }
    r.users[user.ID] = &user
    return nil
}

func TestUserService(t *testing.T) {
    repo := NewFakeUserRepository()
    service := NewUserService(repo)

    err := service.CreateUser("test@example.com")

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if !repo.saveCalled {
        t.Error("expected Save to be called")
    }

    if len(repo.users) != 1 {
        t.Errorf("expected 1 user, got %d", len(repo.users))
    }
}
```

### Anti-Pattern: Not Using Table-Driven Tests

**The violation:**
```go
// ❌ BAD: Repetitive test functions
func TestValidateEmail_Valid(t *testing.T) {
    validator := NewEmailValidator()
    if !validator.Validate("user@example.com") {
        t.Error("expected valid email")
    }
}

func TestValidateEmail_Empty(t *testing.T) {
    validator := NewEmailValidator()
    if validator.Validate("") {
        t.Error("expected invalid email")
    }
}

func TestValidateEmail_NoAt(t *testing.T) {
    validator := NewEmailValidator()
    if validator.Validate("userexample.com") {
        t.Error("expected invalid email")
    }
}

// 10 more similar functions...
```

**Why this is wrong:**
- Code duplication
- Hard to add new cases
- Not idiomatic Go

**The fix:**
```go
// ✅ GOOD: Table-driven tests
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name     string
        email    string
        expected bool
    }{
        {"valid email", "user@example.com", true},
        {"empty email", "", false},
        {"no at symbol", "userexample.com", false},
        {"no domain", "user@", false},
        {"whitespace only", "   ", false},
        {"valid with subdomain", "user@mail.example.com", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            validator := NewEmailValidator()

            result := validator.Validate(tt.email)

            if result != tt.expected {
                t.Errorf("Validate(%q) = %v, expected %v",
                    tt.email, result, tt.expected)
            }
        })
    }
}
```

### Anti-Pattern: Testing Unexported Functions

**The violation:**
```go
// ❌ BAD: Testing unexported implementation details
// validator.go
package validator

func Validate(email string) bool {
    return hasAtSymbol(email) && validDomain(email)
}

func hasAtSymbol(email string) bool {
    return strings.Contains(email, "@")
}

func validDomain(email string) bool {
    parts := strings.Split(email, "@")
    if len(parts) != 2 {
        return false
    }
    return strings.Contains(parts[1], ".")
}

// validator_internal_test.go (in same package!)
package validator

func TestHasAtSymbol(t *testing.T) {
    if !hasAtSymbol("user@example.com") {
        t.Error("expected true")
    }
}

func TestValidDomain(t *testing.T) {
    if !validDomain("user@example.com") {
        t.Error("expected true")
    }
}
```

**Why this is wrong:**
- Testing implementation details
- Couples tests to internal structure
- Refactoring breaks tests

**The fix:**
```go
// ✅ GOOD: Test public API only
// validator_test.go (separate package!)
package validator_test

import (
    "testing"
    "yourmodule/validator"
)

func TestValidate(t *testing.T) {
    tests := []struct {
        name  string
        email string
        valid bool
    }{
        {"valid email", "user@example.com", true},
        {"no at symbol", "userexample.com", false},
        {"no domain dot", "user@examplecom", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := validator.Validate(tt.email)

            if result != tt.valid {
                t.Errorf("Validate(%q) = %v, expected %v",
                    tt.email, result, tt.valid)
            }
        })
    }
}
```

### Anti-Pattern: Not Using t.Helper() in Test Helpers

**The violation:**
```go
// ❌ BAD: Helper without t.Helper()
func assertValid(t *testing.T, validator *EmailValidator, email string) {
    if !validator.Validate(email) {
        t.Errorf("expected %q to be valid", email)
        // Error points to this line, not the test that called it!
    }
}

func TestValidEmails(t *testing.T) {
    validator := NewEmailValidator()
    assertValid(t, validator, "user@example.com")  // Error won't point here
    assertValid(t, validator, "admin@example.com")
}
```

**Why this is wrong:**
- Error messages point to helper function line, not test line
- Hard to debug which assertion failed

**The fix:**
```go
// ✅ GOOD: Use t.Helper()
func assertValid(t *testing.T, validator *EmailValidator, email string) {
    t.Helper()  // Error will point to caller!

    if !validator.Validate(email) {
        t.Errorf("expected %q to be valid", email)
    }
}

func TestValidEmails(t *testing.T) {
    validator := NewEmailValidator()
    assertValid(t, validator, "user@example.com")  // Error points here ✓
    assertValid(t, validator, "admin@example.com")
}
```

### Anti-Pattern: Global State in Tests

**The violation:**
```go
// ❌ BAD: Global variable mutated by tests
var globalDB *sql.DB

func TestUserCreation(t *testing.T) {
    globalDB = setupTestDB()
    defer globalDB.Close()

    // Test uses globalDB
}

func TestUserDeletion(t *testing.T) {
    // Assumes globalDB is set up - flaky!
    // Tests can't run in parallel
}
```

**Why this is wrong:**
- Tests can't run in parallel
- Order-dependent tests
- Flaky and hard to debug

**The fix:**
```go
// ✅ GOOD: Test-scoped state
func TestUserCreation(t *testing.T) {
    t.Parallel()  // Can run in parallel!

    db := setupTestDB(t)
    defer db.Close()

    // Test uses db
}

func TestUserDeletion(t *testing.T) {
    t.Parallel()

    db := setupTestDB(t)
    defer db.Close()

    // Test uses db
}

func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()

    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("failed to open test db: %v", err)
    }

    return db
}
```

### Anti-Pattern: Not Using Subtests for Setup Sharing

**The violation:**
```go
// ❌ BAD: Duplicated setup
func TestUserService_Create(t *testing.T) {
    db := setupDB(t)
    defer db.Close()
    repo := NewRepository(db)
    service := NewUserService(repo)

    // Test create...
}

func TestUserService_Update(t *testing.T) {
    db := setupDB(t)
    defer db.Close()
    repo := NewRepository(db)
    service := NewUserService(repo)

    // Test update...
}

// Duplicated setup in every function
```

**The fix:**
```go
// ✅ GOOD: Use subtests for shared setup
func TestUserService(t *testing.T) {
    // Shared setup
    db := setupDB(t)
    defer db.Close()
    repo := NewRepository(db)
    service := NewUserService(repo)

    t.Run("Create", func(t *testing.T) {
        err := service.Create("test@example.com")
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
    })

    t.Run("Update", func(t *testing.T) {
        err := service.Update("123", "new@example.com")
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
    })

    t.Run("Delete", func(t *testing.T) {
        err := service.Delete("123")
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
    })
}
```

## Quick Reference

| Anti-Pattern | Go-Specific Fix |
|--------------|-----------------|
| Testing mocks | Test public API with real or minimal fake implementations |
| Test-only methods | Extract to `testing/testutil` package |
| Over-mocking | Use simple test doubles instead of mock frameworks |
| Interface pollution | Define interfaces at consumer, not producer |
| testify/mock overuse | Create simple fake implementations |
| Repetitive tests | Use table-driven tests with subtests |
| Testing unexported functions | Test only exported API (use `_test` package) |
| Helper without t.Helper() | Always call `t.Helper()` in test helpers |
| Global state | Use test-scoped state, enable `t.Parallel()` |
| Duplicated setup | Use subtests for shared setup |
| Incomplete test doubles | Implement full interface, return errors for unimplemented methods |
