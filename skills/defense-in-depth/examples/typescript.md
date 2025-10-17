# Defense-in-Depth Validation: TypeScript Examples

> **Note:** This file contains TypeScript-specific patterns. For core defense-in-depth principles, see the main SKILL.md.

## Language-Specific Patterns

TypeScript with Node.js frameworks (Express, NestJS, etc.) provides different validation layers than Rails. Here's how to apply defense-in-depth validation in TypeScript applications.

## The Four Layers in TypeScript

### Layer 1: Entry Point Validation
**Purpose:** Reject obviously invalid input at API boundary

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... proceed
}
```

**Why:** Catches invalid input at the earliest possible point, before it propagates through the system.

### Layer 2: Business Logic Validation
**Purpose:** Ensure data makes sense for this operation

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

**Why:** Different code paths may bypass Layer 1. This catches data that passed Layer 1 but is invalid in this specific context.

### Layer 3: Environment Guards
**Purpose:** Prevent dangerous operations in specific contexts

```typescript
async function gitInit(directory: string) {
  // In tests, refuse git init outside temp directories
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... proceed
}
```

**Why:** Mocks and test fixtures can bypass validation. Environment-specific guards prevent dangerous operations in test/development environments.

### Layer 4: Debug Instrumentation
**Purpose:** Capture context for forensics

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... proceed
}
```

**Why:** When all validation layers fail, instrumentation captures the context needed to diagnose how invalid data reached this point.

## Complete Example: User Email Validation

**Scenario:** Ensure users always have valid, unique emails in a TypeScript/Express API

### Layer 1: Database Schema (with Prisma)

```typescript
// prisma/schema.prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())

  @@index([email])
}
```

**Why:** Database constraints cannot be bypassed by application code.

### Layer 2: Request Validation (with Zod)

```typescript
// src/validators/user.validator.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.string()
    .min(1, 'Email required')
    .email('Invalid email format')
    .transform(val => val.toLowerCase().trim()),
  name: z.string().min(1, 'Name required'),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
```

**Why:** Validates and normalizes data at API entry point before it reaches business logic.

### Layer 3: Service Layer Validation

```typescript
// src/services/user.service.ts
import { PrismaClient } from '@prisma/client';
import type { CreateUserInput } from '../validators/user.validator';

export class UserService {
  constructor(private prisma: PrismaClient) {}

  async createUser(input: CreateUserInput) {
    // Additional business logic validation
    if (!input.email) {
      throw new Error('Email required for user creation');
    }

    const blockedDomains = ['tempmail.com', 'throwaway.email'];
    const domain = input.email.split('@')[1];
    if (blockedDomains.includes(domain)) {
      throw new Error(`Email domain not allowed: ${domain}`);
    }

    // Debug logging
    console.log('Creating user', {
      email: input.email,
      timestamp: new Date().toISOString(),
    });

    try {
      return await this.prisma.user.create({
        data: input,
      });
    } catch (error) {
      if (error.code === 'P2002') { // Prisma unique constraint violation
        throw new Error(`Email already exists: ${input.email}`);
      }
      throw error;
    }
  }
}
```

**Why:**
- Validates business rules (blocked domains)
- Provides friendly error messages for database constraint violations
- Adds debug logging for forensics

### Layer 4: Controller/Route Handler

```typescript
// src/routes/users.routes.ts
import { Router } from 'express';
import { createUserSchema } from '../validators/user.validator';
import { UserService } from '../services/user.service';
import { PrismaClient } from '@prisma/client';

const router = Router();
const prisma = new PrismaClient();
const userService = new UserService(prisma);

router.post('/users', async (req, res) => {
  try {
    // Layer 4: Parse and validate request body
    const input = createUserSchema.parse(req.body);

    // Layer 3: Business logic validation
    const user = await userService.createUser(input);

    res.status(201).json(user);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.errors
      });
    }

    res.status(400).json({
      error: error.message || 'Failed to create user'
    });
  }
});

export default router;
```

**Why:** Orchestrates all validation layers and provides appropriate HTTP responses.

## Why All Four Layers in TypeScript?

Each layer catches what others miss:

**Database constraints** catch:
- Direct SQL bypassing ORM
- Race conditions
- Concurrent creates from multiple instances

**Request validation** catches:
- Invalid formats early
- Type mismatches
- Required field violations

**Service layer** catches:
- Business rules too complex for schemas
- Cross-entity validations
- Context-specific checks

**Controller layer** catches:
- Request-level concerns
- HTTP-specific validation
- Proper error responses

## Testing Each Layer

```typescript
// test/services/user.service.test.ts
describe('UserService', () => {
  it('rejects empty email', async () => {
    const service = new UserService(prisma);

    await expect(
      service.createUser({ email: '', name: 'Test' })
    ).rejects.toThrow('Email required');
  });

  it('rejects blocked domains', async () => {
    const service = new UserService(prisma);

    await expect(
      service.createUser({
        email: 'test@tempmail.com',
        name: 'Test'
      })
    ).rejects.toThrow('Email domain not allowed');
  });

  it('handles database uniqueness constraint', async () => {
    const service = new UserService(prisma);

    await service.createUser({
      email: 'test@example.com',
      name: 'Test'
    });

    await expect(
      service.createUser({
        email: 'test@example.com',
        name: 'Test 2'
      })
    ).rejects.toThrow('Email already exists');
  });
});
```

## Common TypeScript Validation Libraries

### Zod
```typescript
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  age: z.number().int().positive(),
});
```

### class-validator (NestJS)
```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsNotEmpty()
  @IsEmail()
  email: string;

  @IsNotEmpty()
  name: string;
}
```

### Joi
```typescript
import Joi from 'joi';

const schema = Joi.object({
  email: Joi.string().email().required(),
  name: Joi.string().required(),
});
```

## Key Differences from Rails

- **No ActiveRecord callbacks**: Must explicitly call validation in service layer
- **Database agnostic**: Use ORM (Prisma, TypeORM) or query builder for constraints
- **Middleware-based**: Validation often happens in Express/NestJS middleware
- **Type system**: TypeScript types provide compile-time validation layer
- **Schema libraries**: External libraries (Zod, Joi) vs Rails built-in validations
