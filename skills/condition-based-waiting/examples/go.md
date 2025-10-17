# Condition-Based Waiting: Go Examples

> **Note:** This file contains Go-specific patterns. For core condition-based waiting principles, see the main SKILL.md.

## Language-Specific Patterns

### Generic Wait Helper

```go
package testutil

import (
    "fmt"
    "time"
)

// WaitFor polls a condition until it returns true or timeout is reached.
func WaitFor(condition func() bool, description string, timeout time.Duration) error {
    deadline := time.Now().Add(timeout)
    interval := 10 * time.Millisecond

    for {
        if condition() {
            return nil
        }

        if time.Now().After(deadline) {
            return fmt.Errorf("timeout waiting for %s after %v", description, timeout)
        }

        time.Sleep(interval)
    }
}

// WaitForValue polls a condition until it returns a non-nil value.
func WaitForValue[T any](condition func() *T, description string, timeout time.Duration) (T, error) {
    var zero T
    deadline := time.Now().Add(timeout)
    interval := 10 * time.Millisecond

    for {
        if result := condition(); result != nil {
            return *result, nil
        }

        if time.Now().After(deadline) {
            return zero, fmt.Errorf("timeout waiting for %s after %v", description, timeout)
        }

        time.Sleep(interval)
    }
}
```

**Usage:**

```go
// Wait for file to exist
err := WaitFor(
    func() bool {
        _, err := os.Stat("output.txt")
        return err == nil
    },
    "file to be created",
    5*time.Second,
)

// Wait for specific value
user, err := WaitForValue(
    func() *User {
        user, _ := db.GetUser(userID)
        if user != nil && user.Active {
            return user
        }
        return nil
    },
    "user to be active",
    10*time.Second,
)
```

## Channel-Based Waiting

### Waiting for Channel Events

```go
func TestEventProcessing(t *testing.T) {
    events := make(chan Event, 10)
    done := make(chan bool)

    // Start processor
    go ProcessEvents(events, done)

    // Send event
    events <- Event{Type: "test"}

    // ❌ BAD: Guessing at timing
    time.Sleep(100 * time.Millisecond)
    // Check result somehow

    // ✅ GOOD: Wait for done signal
    select {
    case <-done:
        // Success
    case <-time.After(5 * time.Second):
        t.Fatal("timeout waiting for event processing")
    }
}
```

### Waiting for Multiple Channels

```go
func TestMultipleOperations(t *testing.T) {
    ch1 := make(chan Result)
    ch2 := make(chan Result)

    go operation1(ch1)
    go operation2(ch2)

    var result1, result2 Result
    timeout := time.After(10 * time.Second)

    for i := 0; i < 2; i++ {
        select {
        case result1 = <-ch1:
            // First result received
        case result2 = <-ch2:
            // Second result received
        case <-timeout:
            t.Fatal("timeout waiting for operations")
        }
    }

    // Both results received
    assert.Equal(t, "expected1", result1.Value)
    assert.Equal(t, "expected2", result2.Value)
}
```

## Context with Timeout

### Using context.WithTimeout

```go
func TestWithContext(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Start operation
    resultCh := make(chan Result)
    go func() {
        result := performOperation()
        select {
        case resultCh <- result:
        case <-ctx.Done():
            return
        }
    }()

    // Wait for result or timeout
    select {
    case result := <-resultCh:
        assert.Equal(t, "expected", result.Value)
    case <-ctx.Done():
        t.Fatal("operation timed out")
    }
}
```

### Polling with Context

```go
func WaitForConditionCtx(ctx context.Context, condition func() bool, interval time.Duration) error {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        if condition() {
            return nil
        }

        select {
        case <-ticker.C:
            continue
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}

func TestWithPolling(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    service.StartOperation()

    err := WaitForConditionCtx(
        ctx,
        func() bool { return service.Status() == "complete" },
        10*time.Millisecond,
    )
    if err != nil {
        t.Fatalf("operation did not complete: %v", err)
    }
}
```

## Testing HTTP Servers

### Waiting for Server to Be Ready

```go
func TestHTTPServer(t *testing.T) {
    server := &http.Server{Addr: ":8080"}

    // Start server in background
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            t.Errorf("server error: %v", err)
        }
    }()
    defer server.Shutdown(context.Background())

    // ❌ BAD: Guessing at startup time
    time.Sleep(100 * time.Millisecond)

    // ✅ GOOD: Wait for server to accept connections
    err := WaitFor(
        func() bool {
            conn, err := net.Dial("tcp", "localhost:8080")
            if err != nil {
                return false
            }
            conn.Close()
            return true
        },
        "server to be ready",
        5*time.Second,
    )
    if err != nil {
        t.Fatal(err)
    }

    // Server is ready - make requests
    resp, err := http.Get("http://localhost:8080/health")
    assert.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
}
```

## Database Testing

### Waiting for Database Record

```go
func TestUserCreation(t *testing.T) {
    db := setupTestDB(t)

    // Create user asynchronously
    go CreateUserAsync(db, "test@example.com")

    // ❌ BAD: Guessing at timing
    time.Sleep(500 * time.Millisecond)
    user, _ := db.GetUserByEmail("test@example.com")

    // ✅ GOOD: Wait for user to exist
    user, err := WaitForValue(
        func() *User {
            user, _ := db.GetUserByEmail("test@example.com")
            return user
        },
        "user to be created",
        5*time.Second,
    )
    if err != nil {
        t.Fatal(err)
    }

    assert.Equal(t, "test@example.com", user.Email)
}
```

## Goroutine Synchronization

### Using sync.WaitGroup (Wrong Approach)

```go
// ❌ BAD: WaitGroup with timeout is awkward
func TestWithWaitGroup(t *testing.T) {
    var wg sync.WaitGroup
    wg.Add(1)

    go func() {
        defer wg.Done()
        performOperation()
    }()

    // No clean way to timeout with WaitGroup alone
    wg.Wait()
}
```

### Using Channel (Better)

```go
// ✅ GOOD: Channel with timeout
func TestWithChannel(t *testing.T) {
    done := make(chan struct{})

    go func() {
        performOperation()
        close(done)
    }()

    select {
    case <-done:
        // Success
    case <-time.After(5 * time.Second):
        t.Fatal("operation timed out")
    }
}
```

### Helper for Goroutine with Timeout

```go
// RunWithTimeout runs a function in a goroutine with timeout.
func RunWithTimeout(t *testing.T, timeout time.Duration, fn func()) {
    done := make(chan struct{})

    go func() {
        fn()
        close(done)
    }()

    select {
    case <-done:
        return
    case <-time.After(timeout):
        t.Fatal("operation timed out")
    }
}

func TestOperation(t *testing.T) {
    RunWithTimeout(t, 5*time.Second, func() {
        result := performExpensiveOperation()
        assert.Equal(t, "expected", result)
    })
}
```

## Testing Worker Pools

```go
type WorkerPool struct {
    jobs    chan Job
    results chan Result
}

func TestWorkerPool(t *testing.T) {
    pool := NewWorkerPool(5)
    pool.Start()
    defer pool.Stop()

    // Submit jobs
    for i := 0; i < 10; i++ {
        pool.jobs <- Job{ID: i}
    }

    // ❌ BAD: Waiting arbitrary time
    time.Sleep(1 * time.Second)

    // ✅ GOOD: Wait for expected number of results
    results := make([]Result, 0, 10)
    timeout := time.After(10 * time.Second)

    for len(results) < 10 {
        select {
        case result := <-pool.results:
            results = append(results, result)
        case <-timeout:
            t.Fatalf("timeout waiting for results, got %d/10", len(results))
        }
    }

    assert.Equal(t, 10, len(results))
}
```

## Real-World Example: Flaky Test Fix

**BEFORE (flaky):**
```go
func TestVideoTranscoding(t *testing.T) {
    video := createVideo(t, "test.mp4")

    go transcodeVideo(video.ID)

    // Hope it finishes in 2 seconds
    time.Sleep(2 * time.Second)

    result, _ := getVideo(video.ID)
    assert.Equal(t, "completed", result.Status)
    assert.NotEmpty(t, result.TranscodedURL)
}
```

**AFTER (reliable):**
```go
func TestVideoTranscoding(t *testing.T) {
    video := createVideo(t, "test.mp4")

    go transcodeVideo(video.ID)

    // Wait for transcoding to complete
    result, err := WaitForValue(
        func() *Video {
            v, _ := getVideo(video.ID)
            if v != nil && v.Status == "completed" {
                return v
            }
            return nil
        },
        "video transcoding to complete",
        30*time.Second,
    )
    if err != nil {
        t.Fatal(err)
    }

    assert.Equal(t, "completed", result.Status)
    assert.NotEmpty(t, result.TranscodedURL)
}
```

## Testing External Services

### Polling External API

```go
func waitForWebhookProcessed(webhookID string, timeout time.Duration) (*WebhookStatus, error) {
    return WaitForValue(
        func() *WebhookStatus {
            status, err := httpClient.Get(
                fmt.Sprintf("https://api.example.com/webhooks/%s", webhookID),
            )
            if err != nil {
                return nil
            }

            if status.State == "processed" {
                return status
            }
            return nil
        },
        "webhook to be processed",
        timeout,
    )
}

func TestWebhookIntegration(t *testing.T) {
    webhook := triggerWebhook(t, map[string]string{"event": "test"})

    status, err := waitForWebhookProcessed(webhook.ID, 30*time.Second)
    if err != nil {
        t.Fatal(err)
    }

    assert.Equal(t, "processed", status.State)
}
```

## Using testify/assert with Waiting

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestWithTestify(t *testing.T) {
    service := NewService()
    service.Start()

    err := WaitFor(
        func() bool { return service.IsReady() },
        "service to be ready",
        5*time.Second,
    )
    require.NoError(t, err, "service failed to start")

    // Service is ready - run assertions
    assert.Equal(t, "running", service.Status())
}
```

## Common Pitfalls

### Pitfall 1: Forgetting to Close Channels

```go
// ❌ BAD: Channel never closes, goroutine leaks
func TestChannelLeak(t *testing.T) {
    ch := make(chan Result)

    go func() {
        result := compute()
        ch <- result
        // Channel never closed!
    }()

    select {
    case result := <-ch:
        // Got result but goroutine still running
    case <-time.After(5 * time.Second):
        t.Fatal("timeout")
    }
}

// ✅ GOOD: Close channel when done
func TestChannelNoLeak(t *testing.T) {
    ch := make(chan Result)

    go func() {
        defer close(ch)
        result := compute()
        ch <- result
    }()

    select {
    case result, ok := <-ch:
        if !ok {
            t.Fatal("channel closed without result")
        }
        assert.Equal(t, "expected", result.Value)
    case <-time.After(5 * time.Second):
        t.Fatal("timeout")
    }
}
```

### Pitfall 2: Race Conditions in Polling

```go
// ❌ BAD: Reading without synchronization
type Service struct {
    status string
}

func (s *Service) Status() string {
    return s.status  // Race condition!
}

func TestRaceCondition(t *testing.T) {
    service := &Service{}
    go service.Start()  // Writes to status

    err := WaitFor(
        func() bool { return service.Status() == "ready" },  // Reads status
        "service to be ready",
        5*time.Second,
    )
}

// ✅ GOOD: Use mutex or channels
type Service struct {
    mu     sync.RWMutex
    status string
}

func (s *Service) Status() string {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.status
}
```

### Pitfall 3: Polling Too Frequently

```go
// ❌ BAD: Tight loop wastes CPU
func waitForFileBad(path string) error {
    deadline := time.Now().Add(5 * time.Second)
    for time.Now().Before(deadline) {
        if _, err := os.Stat(path); err == nil {
            return nil
        }
        // No sleep - spins CPU!
    }
    return fmt.Errorf("timeout")
}

// ✅ GOOD: Sleep between checks
func waitForFileGood(path string) error {
    deadline := time.Now().Add(5 * time.Second)
    for time.Now().Before(deadline) {
        if _, err := os.Stat(path); err == nil {
            return nil
        }
        time.Sleep(10 * time.Millisecond)
    }
    return fmt.Errorf("timeout")
}
```
