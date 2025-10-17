# Test-Driven Development: TypeScript Examples

> **Note:** This file contains TypeScript-specific patterns. For core TDD principles, see the main SKILL.md.

## Language-Specific Patterns

### Test Structure

TypeScript tests typically use Jest or Vitest:

```typescript
// Test files must end with .test.ts or .spec.ts
// email-validator.test.ts

import { describe, test, expect, beforeEach } from '@jest/globals';
import { EmailValidator } from './email-validator';

describe('EmailValidator', () => {
  let validator: EmailValidator;

  beforeEach(() => {
    validator = new EmailValidator();
  });

  test('valid returns true for valid email', () => {
    expect(validator.valid('user@example.com')).toBe(true);
  });

  test('valid returns false for empty email', () => {
    expect(validator.valid('')).toBe(false);
  });

  test('normalize removes whitespace', () => {
    expect(EmailValidator.normalize('  user@example.com  ')).toBe('user@example.com');
  });
});
```

**Key patterns:**
- Instance methods: `test('method behavior', () => { ... })`
- Static methods: Call directly on class
- Setup: `beforeEach` and `afterEach`
- Each test: Setup → Action → Assertion

### Assertions

```typescript
// Jest/Vitest assertions
expect(actual).toBe(expected);                  // Strict equality (===)
expect(actual).toEqual(expected);               // Deep equality
expect(collection).toHaveLength(0);             // Empty collections
expect(obj).toBeInstanceOf(MyClass);            // Type checking
expect(predicate()).toBeTruthy();               // Boolean check
expect(value).not.toBe(expected);               // Inequality
expect(value).not.toBeNull();                   // Not null/undefined
```

### RED - Write Failing Test

<Good>
```typescript
test('retries failed operations 3 times', async () => {
  let attempts = 0;
  const operation = () => {
    attempts++;
    if (attempts < 3) throw new Error('fail');
    return 'success';
  };

  const result = await retryOperation(operation);

  expect(result).toBe('success');
  expect(attempts).toBe(3);
});
```
Clear name, tests real behavior, one thing
</Good>

<Bad>
```typescript
test('retry works', async () => {
  const mock = jest.fn()
    .mockRejectedValueOnce(new Error())
    .mockRejectedValueOnce(new Error())
    .mockResolvedValueOnce('success');
  await retryOperation(mock);
  expect(mock).toHaveBeenCalledTimes(3);
});
```
Vague name, tests mock not code
</Bad>

### GREEN - Minimal Code

<Good>
```typescript
async function retryOperation<T>(fn: () => Promise<T>): Promise<T> {
  for (let i = 0; i < 3; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === 2) throw e;
    }
  }
  throw new Error('unreachable');
}
```
Just enough to pass
</Good>

<Bad>
```typescript
async function retryOperation<T>(
  fn: () => Promise<T>,
  options?: {
    maxRetries?: number;
    backoff?: 'linear' | 'exponential';
    onRetry?: (attempt: number) => void;
  }
): Promise<T> {
  // YAGNI
}
```
Over-engineered
</Bad>

### Mocking with Jest

```typescript
import { jest } from '@jest/globals';

// CORRECT - Mock external dependencies only
test('service calls repository', async () => {
  const mockRepo = {
    save: jest.fn().mockResolvedValue(true)
  };

  const service = new UserService(mockRepo);
  const result = await service.createUser('test@example.com');

  expect(mockRepo.save).toHaveBeenCalledOnce();
  expect(result).toBe(true);
});

// WRONG - Testing mock behavior (see Testing Anti-Patterns skill)
test('retry works', async () => {
  const mock = jest.fn()
    .mockRejectedValueOnce(new Error())
    .mockResolvedValueOnce('success');

  await retryOperation(mock);

  expect(mock).toHaveBeenCalled(); // This tests the mock, not your code
});
```

### Running Tests

```bash
# Run all tests
npm test

# Run single test file
npm test path/to/test.test.ts

# Run specific test
npm test -- --testNamePattern="retries failed operations"

# Run with coverage
npm test -- --coverage

# Watch mode
npm test -- --watch
```

### Coverage Requirements

```bash
# Check coverage (must be ≥95% branch)
npm test -- --coverage --coverageThreshold='{"branches": 95}'

# Generate HTML report
npm test -- --coverage --coverageReporters=html
# Open coverage/index.html
```

### Complete Example: Bug Fix

**Bug:** Empty email accepted by validator

**RED**
```typescript
test('rejects empty email', async () => {
  const result = await submitForm({ email: '' });
  expect(result.error).toBe('Email required');
});
```

**Verify RED**
```bash
$ npm test
FAIL: expected 'Email required', got undefined
```

**GREEN**
```typescript
function submitForm(data: FormData) {
  if (!data.email?.trim()) {
    return { error: 'Email required' };
  }
  // ...
}
```

**Verify GREEN**
```bash
$ npm test
PASS
```

**REFACTOR**
Extract validation for multiple fields if needed.

### TypeScript-Specific Features

**Type safety in tests:**
```typescript
// Tests catch type errors
test('handles typed responses', async () => {
  const result: ApiResponse = await fetchData();

  // TypeScript ensures these properties exist
  expect(result.status).toBe(200);
  expect(result.data).toBeDefined();
});
```

**Async/await patterns:**
```typescript
test('handles async operations', async () => {
  // All async operations must be awaited
  const result = await asyncOperation();
  expect(result).toBe('done');
});

test('handles promise rejection', async () => {
  // Test async errors
  await expect(failingOperation()).rejects.toThrow('error message');
});
```

**Test organization with describe blocks:**
```typescript
describe('EmailValidator', () => {
  describe('valid', () => {
    test('returns true for valid email', () => {
      // Valid email tests
    });

    test('returns false for invalid email', () => {
      // Invalid email tests
    });
  });

  describe('normalize', () => {
    test('removes whitespace', () => {
      // Normalization tests
    });
  });
});
```
