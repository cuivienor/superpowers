# Testing Anti-Patterns: TypeScript Examples

> **Note:** This file contains TypeScript-specific anti-patterns. For core testing principles, see the main SKILL.md.

## Language-Specific Anti-Patterns

### Anti-Pattern 1: Testing Mock Behavior (React Testing Library)

**The violation:**
```typescript
// ❌ BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**Why this is wrong:**
- You're verifying the mock works, not that the component works
- Test passes when mock is present, fails when it's not
- Tells you nothing about real behavior

**The fix:**
```typescript
// ✅ GOOD: Test real component or don't mock it
test('renders sidebar', () => {
  render(<Page />);  // Don't mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### Anti-Pattern 2: Test-Only Methods in Production Classes

**The violation:**
```typescript
// ❌ BAD: destroy() only used in tests
class Session {
  async destroy() {  // Looks like production API!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// In tests
afterEach(() => session.destroy());
```

**Why this is wrong:**
- Production class polluted with test-only code
- Dangerous if accidentally called in production
- Violates YAGNI and separation of concerns

**The fix:**
```typescript
// ✅ GOOD: Test utilities handle test cleanup
// Session has no destroy() - it's stateless in production

// In test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// In tests
afterEach(() => cleanupSession(session));
```

### Anti-Pattern 3: Mocking Without Understanding (Vitest)

**The violation:**
```typescript
// ❌ BAD: Mock breaks test logic
test('detects duplicate server', () => {
  // Mock prevents config write that test depends on!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // Should throw - but won't!
});
```

**Why this is wrong:**
- Mocked method had side effect test depended on (writing config)
- Over-mocking to "be safe" breaks actual behavior
- Test passes for wrong reason or fails mysteriously

**The fix:**
```typescript
// ✅ GOOD: Mock at correct level
test('detects duplicate server', () => {
  // Mock the slow part, preserve behavior test needs
  vi.mock('MCPServerManager'); // Just mock slow server startup

  await addServer(config);  // Config written
  await addServer(config);  // Duplicate detected ✓
});
```

### Anti-Pattern 4: Incomplete Mocks

**The violation:**
```typescript
// ❌ BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**The fix:**
```typescript
// ✅ GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

## TypeScript-Specific Anti-Patterns

### Anti-Pattern: Over-Using `any` to Fix Test Type Errors

**The violation:**
```typescript
// ❌ BAD: Silencing type errors with any
test('processes user data', () => {
  const mockUser: any = { name: 'Alice' };  // Missing required fields
  const result = processUser(mockUser);  // Should fail at compile time!
  expect(result).toBe('Alice');
});
```

**Why this is wrong:**
- Type system would catch incomplete mock
- `any` defeats TypeScript's safety
- Test compiles but may fail at runtime

**The fix:**
```typescript
// ✅ GOOD: Use proper types
test('processes user data', () => {
  const mockUser: User = {
    id: '123',
    name: 'Alice',
    email: 'alice@example.com',
    createdAt: new Date()
  };
  const result = processUser(mockUser);
  expect(result).toBe('Alice');
});

// OR use a factory function
test('processes user data', () => {
  const mockUser = createTestUser({ name: 'Alice' });
  const result = processUser(mockUser);
  expect(result).toBe('Alice');
});
```

### Anti-Pattern: Manual Mock Modules Instead of Dependency Injection

**The violation:**
```typescript
// ❌ BAD: Hard-coded import requires manual mocking
// api.ts
import { httpClient } from './http-client';

export async function fetchUser(id: string) {
  return httpClient.get(`/users/${id}`);
}

// api.test.ts
vi.mock('./http-client', () => ({
  httpClient: { get: vi.fn() }
}));
```

**Why this is wrong:**
- Hard to test without mocking framework magic
- Unclear what dependencies exist
- Mock setup fragile and framework-dependent

**The fix:**
```typescript
// ✅ GOOD: Use dependency injection
// api.ts
export async function fetchUser(id: string, client = httpClient) {
  return client.get(`/users/${id}`);
}

// api.test.ts
test('fetches user by id', async () => {
  const mockClient = {
    get: vi.fn().mockResolvedValue({ id: '123', name: 'Alice' })
  };

  const result = await fetchUser('123', mockClient);

  expect(result).toEqual({ id: '123', name: 'Alice' });
  expect(mockClient.get).toHaveBeenCalledWith('/users/123');
});
```

### Anti-Pattern: Testing Implementation Details with Jest Spies

**The violation:**
```typescript
// ❌ BAD: Spying on internal methods
test('form submission', () => {
  const form = new ContactForm();
  const validateSpy = vi.spyOn(form, 'validateEmail');

  form.submit({ email: 'test@example.com' });

  expect(validateSpy).toHaveBeenCalled();  // Testing implementation!
});
```

**Why this is wrong:**
- Couples test to implementation
- Refactoring breaks tests
- Doesn't verify actual behavior

**The fix:**
```typescript
// ✅ GOOD: Test observable behavior
test('form submission accepts valid email', () => {
  const form = new ContactForm();

  const result = form.submit({ email: 'test@example.com' });

  expect(result.success).toBe(true);
});

test('form submission rejects invalid email', () => {
  const form = new ContactForm();

  const result = form.submit({ email: 'invalid' });

  expect(result.success).toBe(false);
  expect(result.error).toBe('Invalid email');
});
```

### Anti-Pattern: Async Test Without Proper Awaiting

**The violation:**
```typescript
// ❌ BAD: Not awaiting async operations
test('loads user data', () => {
  loadUserData('123');  // Returns Promise, but not awaited
  expect(userStore.user).toEqual({ id: '123', name: 'Alice' });
  // Assertion runs before data loads!
});
```

**Why this is wrong:**
- Test runs before async operation completes
- False positives or race conditions
- Flaky tests

**The fix:**
```typescript
// ✅ GOOD: Properly await async operations
test('loads user data', async () => {
  await loadUserData('123');
  expect(userStore.user).toEqual({ id: '123', name: 'Alice' });
});

// OR use waitFor for UI updates
test('displays user data', async () => {
  render(<UserProfile userId="123" />);

  await waitFor(() => {
    expect(screen.getByText('Alice')).toBeInTheDocument();
  });
});
```

## React Testing Library Specific Anti-Patterns

### Anti-Pattern: Querying by Implementation Details

**The violation:**
```typescript
// ❌ BAD: Using data-testid for everything
test('renders user profile', () => {
  render(<UserProfile user={user} />);
  expect(screen.getByTestId('user-name')).toHaveTextContent('Alice');
  expect(screen.getByTestId('user-email')).toHaveTextContent('alice@example.com');
});
```

**Why this is wrong:**
- Pollutes production code with test IDs
- Doesn't test accessibility
- Not how users interact with UI

**The fix:**
```typescript
// ✅ GOOD: Use semantic queries
test('renders user profile', () => {
  render(<UserProfile user={user} />);
  expect(screen.getByRole('heading', { name: 'Alice' })).toBeInTheDocument();
  expect(screen.getByText('alice@example.com')).toBeInTheDocument();
});

// Only use testid as last resort for non-semantic elements
```

### Anti-Pattern: Testing React Component Internals

**The violation:**
```typescript
// ❌ BAD: Accessing component state
test('counter increments', () => {
  const { container } = render(<Counter />);
  const component = container.querySelector('.counter');

  fireEvent.click(screen.getByRole('button'));

  expect(component.state.count).toBe(1);  // Accessing internals!
});
```

**Why this is wrong:**
- Tests implementation, not behavior
- Couples test to internal structure
- Users don't care about state

**The fix:**
```typescript
// ✅ GOOD: Test observable output
test('counter increments', () => {
  render(<Counter />);

  fireEvent.click(screen.getByRole('button', { name: 'Increment' }));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

## Quick Reference

| Anti-Pattern | TypeScript-Specific Fix |
|--------------|-------------------------|
| Testing mocks | Use React Testing Library semantic queries |
| Test-only methods | Extract to `test-utils/` directory |
| Over-mocking | Use dependency injection, avoid `vi.mock` when possible |
| Incomplete mocks | Use TypeScript interfaces to enforce completeness |
| Type errors in tests | Define test data factories with proper types |
| Manual module mocks | Inject dependencies at function/constructor level |
| Testing implementation | Test public API and observable behavior only |
| Async without await | Always `await` or use `waitFor` for async operations |
