# Defense-in-Depth Validation: Go Examples

> **Note:** This file contains Go-specific patterns. For core defense-in-depth principles, see the main SKILL.md.

## Language-Specific Patterns

Go's explicit error handling and minimal magic require a different approach to validation layers. Here's how to apply defense-in-depth validation in Go applications.

## Complete Example: User Email Validation

**Scenario:** Ensure users always have valid, unique emails in a Go web service

### Layer 1: Database Constraints

Using PostgreSQL with database/sql or pgx:

```go
// migrations/001_create_users.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(254) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT email_not_empty CHECK (email != ''),
    CONSTRAINT email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
```

**Why:** Database constraints protect against:
- Any code bypassing application logic
- Bulk operations
- Race conditions from concurrent requests
- Migration scripts

### Layer 2: Domain Model Validation

```go
// internal/domain/user.go
package domain

import (
    "errors"
    "net/mail"
    "strings"
    "time"
)

var (
    ErrEmailRequired     = errors.New("email is required")
    ErrEmailInvalid      = errors.New("email format is invalid")
    ErrEmailDomainBlocked = errors.New("email domain is not allowed")
)

// User represents a user in the system
type User struct {
    ID        int64
    Email     string
    Name      string
    CreatedAt time.Time
}

// NewUser creates a new user with validation
func NewUser(email, name string) (*User, error) {
    user := &User{
        Email:     email,
        Name:      name,
        CreatedAt: time.Now(),
    }

    if err := user.Validate(); err != nil {
        return nil, err
    }

    return user, nil
}

// Validate checks if the user data is valid
func (u *User) Validate() error {
    // Normalize email
    u.Email = strings.TrimSpace(strings.ToLower(u.Email))

    // Check required
    if u.Email == "" {
        return ErrEmailRequired
    }

    // Check format using stdlib
    if _, err := mail.ParseAddress(u.Email); err != nil {
        return ErrEmailInvalid
    }

    // Check blocked domains
    domain := u.emailDomain()
    if isBlockedDomain(domain) {
        return ErrEmailDomainBlocked
    }

    return nil
}

func (u *User) emailDomain() string {
    parts := strings.Split(u.Email, "@")
    if len(parts) != 2 {
        return ""
    }
    return parts[1]
}

func isBlockedDomain(domain string) bool {
    blocked := []string{"tempmail.com", "throwaway.email", "guerrillamail.com"}
    for _, b := range blocked {
        if domain == b {
            return true
        }
    }
    return false
}
```

**Why:** Domain model validation ensures business rules are enforced regardless of how the object is created.

### Layer 3: Repository Layer with Validation

Using sqlc for type-safe SQL:

```go
// internal/repository/user_repository.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "log"

    "github.com/lib/pq"
    "myapp/internal/domain"
)

var (
    ErrEmailAlreadyExists = errors.New("email already exists")
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

// Create creates a new user with validation
func (r *UserRepository) Create(ctx context.Context, user *domain.User) error {
    // Layer 3: Pre-insert validation
    if user.Email == "" {
        return domain.ErrEmailRequired
    }

    // Debug logging
    log.Printf("Creating user: email=%s", user.Email)

    query := `
        INSERT INTO users (email, name, created_at)
        VALUES ($1, $2, $3)
        RETURNING id
    `

    err := r.db.QueryRowContext(
        ctx,
        query,
        user.Email,
        user.Name,
        user.CreatedAt,
    ).Scan(&user.ID)

    if err != nil {
        // Layer 1: Handle database constraint violations
        if pqErr, ok := err.(*pq.Error); ok {
            switch pqErr.Code {
            case "23505": // unique_violation
                return ErrEmailAlreadyExists
            case "23514": // check_violation
                return domain.ErrEmailInvalid
            }
        }
        return fmt.Errorf("failed to create user: %w", err)
    }

    return nil
}

// FindByEmail finds a user by email
func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*domain.User, error) {
    // Normalize before query
    email = strings.TrimSpace(strings.ToLower(email))

    query := `
        SELECT id, email, name, created_at
        FROM users
        WHERE email = $1
    `

    user := &domain.User{}
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &user.ID,
        &user.Email,
        &user.Name,
        &user.CreatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("failed to find user: %w", err)
    }

    return user, nil
}
```

**Why:**
- Translates database errors into domain errors
- Adds logging for debugging
- Normalizes data before queries
- Pre-insert validation as safety check

### Layer 4: HTTP Handler Validation

```go
// internal/handlers/user_handler.go
package handlers

import (
    "encoding/json"
    "errors"
    "log"
    "net/http"

    "myapp/internal/domain"
    "myapp/internal/repository"
)

type UserHandler struct {
    repo *repository.UserRepository
}

func NewUserHandler(repo *repository.UserRepository) *UserHandler {
    return &UserHandler{repo: repo}
}

type CreateUserRequest struct {
    Email string `json:"email"`
    Name  string `json:"name"`
}

type ErrorResponse struct {
    Error string `json:"error"`
}

// CreateUser handles POST /users
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // Layer 4: HTTP-level validation
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // Layer 4: Request-level validation
    if req.Email == "" {
        respondError(w, "Email is required", http.StatusBadRequest)
        return
    }

    // Debug logging with context
    log.Printf(
        "CreateUser request: email=%s, ip=%s, user_agent=%s",
        req.Email,
        r.RemoteAddr,
        r.UserAgent(),
    )

    // Layer 2: Create domain object with validation
    user, err := domain.NewUser(req.Email, req.Name)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrEmailRequired):
            respondError(w, "Email is required", http.StatusBadRequest)
        case errors.Is(err, domain.ErrEmailInvalid):
            respondError(w, "Invalid email format", http.StatusBadRequest)
        case errors.Is(err, domain.ErrEmailDomainBlocked):
            respondError(w, "Email domain not allowed", http.StatusBadRequest)
        default:
            respondError(w, "Validation failed", http.StatusBadRequest)
        }
        return
    }

    // Layer 3: Repository validation and database constraints
    if err := h.repo.Create(r.Context(), user); err != nil {
        switch {
        case errors.Is(err, repository.ErrEmailAlreadyExists):
            respondError(w, "Email already exists", http.StatusConflict)
        default:
            log.Printf("Failed to create user: %v", err)
            respondError(w, "Internal server error", http.StatusInternalServerError)
        }
        return
    }

    // Success
    respondJSON(w, user, http.StatusCreated)
}

func respondError(w http.ResponseWriter, message string, status int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{Error: message})
}

func respondJSON(w http.ResponseWriter, data interface{}, status int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

**Why:**
- HTTP-specific validation (method, content-type)
- Request parsing and early rejection
- Proper error responses with status codes
- Debug logging with request context

## Why All Four Layers in Go?

Each layer catches what others miss:

**Database constraints** catch:
- Raw SQL queries bypassing application logic
- Race conditions between checks and inserts
- Any code path that skips validation

**Domain model validation** catches:
- Business logic violations
- Invalid object construction
- Data normalization needs

**Repository validation** catches:
- Pre-persistence checks
- Friendly error messages for DB errors
- Query-level concerns

**Handler validation** catches:
- HTTP-specific concerns
- Request parsing failures
- Client-friendly error responses

## Testing Each Layer

```go
// internal/domain/user_test.go
package domain_test

import (
    "testing"

    "myapp/internal/domain"
)

func TestUser_Validate_EmptyEmail(t *testing.T) {
    user := &domain.User{Email: "", Name: "Test"}

    err := user.Validate()

    if err != domain.ErrEmailRequired {
        t.Errorf("expected ErrEmailRequired, got %v", err)
    }
}

func TestUser_Validate_InvalidFormat(t *testing.T) {
    user := &domain.User{Email: "notanemail", Name: "Test"}

    err := user.Validate()

    if err != domain.ErrEmailInvalid {
        t.Errorf("expected ErrEmailInvalid, got %v", err)
    }
}

func TestUser_Validate_BlockedDomain(t *testing.T) {
    user := &domain.User{Email: "test@tempmail.com", Name: "Test"}

    err := user.Validate()

    if err != domain.ErrEmailDomainBlocked {
        t.Errorf("expected ErrEmailDomainBlocked, got %v", err)
    }
}

func TestUser_Validate_NormalizesEmail(t *testing.T) {
    user := &domain.User{Email: "  TEST@EXAMPLE.COM  ", Name: "Test"}

    err := user.Validate()

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    expected := "test@example.com"
    if user.Email != expected {
        t.Errorf("expected email %q, got %q", expected, user.Email)
    }
}
```

```go
// internal/repository/user_repository_test.go
package repository_test

import (
    "context"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/repository"
    "myapp/internal/testutil"
)

func TestUserRepository_Create_DuplicateEmail(t *testing.T) {
    db := testutil.SetupTestDB(t)
    defer db.Close()

    repo := repository.NewUserRepository(db)
    ctx := context.Background()

    // Create first user
    user1, _ := domain.NewUser("test@example.com", "Test 1")
    if err := repo.Create(ctx, user1); err != nil {
        t.Fatalf("failed to create first user: %v", err)
    }

    // Try to create duplicate
    user2, _ := domain.NewUser("test@example.com", "Test 2")
    err := repo.Create(ctx, user2)

    if err != repository.ErrEmailAlreadyExists {
        t.Errorf("expected ErrEmailAlreadyExists, got %v", err)
    }
}
```

## Using Validation Libraries

### go-playground/validator

```go
import "github.com/go-playground/validator/v10"

type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email"`
    Name  string `json:"name" validate:"required,min=1,max=100"`
}

func validateRequest(req CreateUserRequest) error {
    validate := validator.New()
    return validate.Struct(req)
}
```

### ozzo-validation

```go
import validation "github.com/go-ozzo/ozzo-validation/v4"
import "github.com/go-ozzo/ozzo-validation/v4/is"

func (u User) Validate() error {
    return validation.ValidateStruct(&u,
        validation.Field(&u.Email,
            validation.Required,
            is.Email,
        ),
        validation.Field(&u.Name,
            validation.Required,
            validation.Length(1, 100),
        ),
    )
}
```

## Advanced Patterns

### Validation Middleware

```go
// middleware/validation.go
func ValidateRequest(validate func(r *http.Request) error) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if err := validate(r); err != nil {
                respondError(w, err.Error(), http.StatusBadRequest)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
func validateCreateUser(r *http.Request) error {
    if r.Method != http.MethodPost {
        return errors.New("method not allowed")
    }
    if r.Header.Get("Content-Type") != "application/json" {
        return errors.New("content-type must be application/json")
    }
    return nil
}

handler := ValidateRequest(validateCreateUser)(http.HandlerFunc(createUserHandler))
```

### Context-Based Validation

```go
// Validate based on context (e.g., test environment)
func (r *UserRepository) Create(ctx context.Context, user *domain.User) error {
    // Environment-specific guard
    if testing.Testing() {
        if !strings.Contains(user.Email, "@test.com") {
            return errors.New("test emails must use @test.com domain")
        }
    }

    // ... proceed
}
```

### Validation Result Type

```go
type ValidationResult struct {
    Valid  bool
    Errors map[string][]string
}

func (u *User) ValidateDetailed() ValidationResult {
    result := ValidationResult{
        Valid:  true,
        Errors: make(map[string][]string),
    }

    if u.Email == "" {
        result.Valid = false
        result.Errors["email"] = append(result.Errors["email"], "Email is required")
    }

    if _, err := mail.ParseAddress(u.Email); err != nil {
        result.Valid = false
        result.Errors["email"] = append(result.Errors["email"], "Invalid email format")
    }

    return result
}
```

## Key Differences from Rails/Django

- **No ORM magic**: Must explicitly validate before database operations
- **Explicit error handling**: Each layer must handle and translate errors
- **Interface-based design**: Validation often via interfaces (e.g., `Validator` interface)
- **Minimal reflection**: Struct tags for validation libraries, but generally prefer explicit code
- **Database constraints via SQL**: Define constraints in migration SQL files
- **No callbacks**: Must explicitly call validation methods
- **Context-aware**: Use `context.Context` to pass request-scoped data through layers
