# Condition-Based Waiting: TypeScript Examples

> **Note:** This file contains TypeScript-specific patterns. For core condition-based waiting principles, see the main SKILL.md.

## Language-Specific Patterns

### Complete Implementation

This is a complete implementation of condition-based waiting utilities from a real debugging session (2025-10-03) that fixed 15 flaky tests by replacing arbitrary timeouts.

```typescript
// Complete implementation of condition-based waiting utilities
// From: Lace test infrastructure improvements (2025-10-03)
// Context: Fixed 15 flaky tests by replacing arbitrary timeouts

import type { ThreadManager } from '~/threads/thread-manager';
import type { LaceEvent, LaceEventType } from '~/threads/types';

/**
 * Wait for a specific event type to appear in thread
 *
 * @param threadManager - The thread manager to query
 * @param threadId - Thread to check for events
 * @param eventType - Type of event to wait for
 * @param timeoutMs - Maximum time to wait (default 5000ms)
 * @returns Promise resolving to the first matching event
 *
 * Example:
 *   await waitForEvent(threadManager, agentThreadId, 'TOOL_RESULT');
 */
export function waitForEvent(
  threadManager: ThreadManager,
  threadId: string,
  eventType: LaceEventType,
  timeoutMs = 5000
): Promise<LaceEvent> {
  return new Promise((resolve, reject) => {
    const startTime = Date.now();

    const check = () => {
      const events = threadManager.getEvents(threadId);
      const event = events.find((e) => e.type === eventType);

      if (event) {
        resolve(event);
      } else if (Date.now() - startTime > timeoutMs) {
        reject(new Error(`Timeout waiting for ${eventType} event after ${timeoutMs}ms`));
      } else {
        setTimeout(check, 10); // Poll every 10ms for efficiency
      }
    };

    check();
  });
}

/**
 * Wait for a specific number of events of a given type
 *
 * @param threadManager - The thread manager to query
 * @param threadId - Thread to check for events
 * @param eventType - Type of event to wait for
 * @param count - Number of events to wait for
 * @param timeoutMs - Maximum time to wait (default 5000ms)
 * @returns Promise resolving to all matching events once count is reached
 *
 * Example:
 *   // Wait for 2 AGENT_MESSAGE events (initial response + continuation)
 *   await waitForEventCount(threadManager, agentThreadId, 'AGENT_MESSAGE', 2);
 */
export function waitForEventCount(
  threadManager: ThreadManager,
  threadId: string,
  eventType: LaceEventType,
  count: number,
  timeoutMs = 5000
): Promise<LaceEvent[]> {
  return new Promise((resolve, reject) => {
    const startTime = Date.now();

    const check = () => {
      const events = threadManager.getEvents(threadId);
      const matchingEvents = events.filter((e) => e.type === eventType);

      if (matchingEvents.length >= count) {
        resolve(matchingEvents);
      } else if (Date.now() - startTime > timeoutMs) {
        reject(
          new Error(
            `Timeout waiting for ${count} ${eventType} events after ${timeoutMs}ms (got ${matchingEvents.length})`
          )
        );
      } else {
        setTimeout(check, 10);
      }
    };

    check();
  });
}

/**
 * Wait for an event matching a custom predicate
 * Useful when you need to check event data, not just type
 *
 * @param threadManager - The thread manager to query
 * @param threadId - Thread to check for events
 * @param predicate - Function that returns true when event matches
 * @param description - Human-readable description for error messages
 * @param timeoutMs - Maximum time to wait (default 5000ms)
 * @returns Promise resolving to the first matching event
 *
 * Example:
 *   // Wait for TOOL_RESULT with specific ID
 *   await waitForEventMatch(
 *     threadManager,
 *     agentThreadId,
 *     (e) => e.type === 'TOOL_RESULT' && e.data.id === 'call_123',
 *     'TOOL_RESULT with id=call_123'
 *   );
 */
export function waitForEventMatch(
  threadManager: ThreadManager,
  threadId: string,
  predicate: (event: LaceEvent) => boolean,
  description: string,
  timeoutMs = 5000
): Promise<LaceEvent> {
  return new Promise((resolve, reject) => {
    const startTime = Date.now();

    const check = () => {
      const events = threadManager.getEvents(threadId);
      const event = events.find(predicate);

      if (event) {
        resolve(event);
      } else if (Date.now() - startTime > timeoutMs) {
        reject(new Error(`Timeout waiting for ${description} after ${timeoutMs}ms`));
      } else {
        setTimeout(check, 10);
      }
    };

    check();
  });
}
```

### Generic Wait Helper

For general-purpose condition polling:

```typescript
async function waitFor<T>(
  condition: () => T | undefined | null | false,
  description: string,
  timeoutMs = 5000
): Promise<T> {
  const startTime = Date.now();

  while (true) {
    const result = condition();
    if (result) return result;

    if (Date.now() - startTime > timeoutMs) {
      throw new Error(`Timeout waiting for ${description} after ${timeoutMs}ms`);
    }

    await new Promise(r => setTimeout(r, 10)); // Poll every 10ms
  }
}
```

## Usage Example: Before and After

**BEFORE (flaky):**
```typescript
const messagePromise = agent.sendMessage('Execute tools');
await new Promise(r => setTimeout(r, 300)); // Hope tools start in 300ms
agent.abort();
await messagePromise;
await new Promise(r => setTimeout(r, 50));  // Hope results arrive in 50ms
expect(toolResults.length).toBe(2);         // Fails randomly
```

**AFTER (reliable):**
```typescript
const messagePromise = agent.sendMessage('Execute tools');
await waitForEventCount(threadManager, threadId, 'TOOL_CALL', 2); // Wait for tools to start
agent.abort();
await messagePromise;
await waitForEventCount(threadManager, threadId, 'TOOL_RESULT', 2); // Wait for results
expect(toolResults.length).toBe(2); // Always succeeds
```

**Result:** 60% pass rate → 100%, 40% faster execution

## TypeScript-Specific Features

### Promise-based polling

TypeScript/JavaScript naturally supports Promise-based async/await, making condition polling clean:

```typescript
// Wait for DOM element
await waitFor(() => document.querySelector('.loaded'), 'element to be loaded');

// Wait for state change
await waitFor(() => store.getState().ready, 'store to be ready');

// Wait for async operation
await waitFor(async () => await fetchStatus() === 'complete', 'operation to complete');
```

### Testing Libraries

**Jest/Vitest:** Built-in `waitFor` in testing libraries:

```typescript
import { waitFor } from '@testing-library/react';

await waitFor(() => expect(screen.getByText('Loaded')).toBeInTheDocument());
```

**Playwright:** Built-in waiting for web elements:

```typescript
await page.waitForSelector('.loaded');
await page.waitForFunction(() => window.myApp.ready === true);
```
