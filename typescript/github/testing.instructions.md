---
description: "Testing standards and best practices for TypeScript projects using Vitest (preferred) or Jest, with integration tests via Testcontainers."
---

# Testing Standards — TypeScript / Node.js

## General Principles

- Every feature must have tests **before** being merged.
- Follow the **Testing Trophy** model: many integration tests, fewer unit tests for pure logic, contracts for external services.
- Tests are **documentation** — test names must describe the behavior, not the implementation.
- Tests must be **deterministic** — no flaky tests, no external dependencies, no time-sensitive logic without mocking.
- **Security tests** are mandatory for every endpoint and user input handler.
- **Coverage thresholds**: statements ≥ 80%, branches ≥ 75%, functions ≥ 80%.

---

## Test Runner — Vitest (preferred)

Vitest is preferred for TypeScript projects due to native ESM support, faster execution, and built-in TypeScript support without extra configuration.

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      thresholds: { statements: 80, branches: 75, functions: 80, lines: 80 },
      exclude: ['**/*.d.ts', '**/*.config.*', '**/index.ts', 'dist/**'],
    },
    include: ['src/**/*.test.ts', 'src/**/*.spec.ts'],
    exclude: ['src/**/*.integration.test.ts'], // separate config for integration
  },
});
```

---

## Test File Organization

```
src/
├── users/
│   ├── user.service.ts
│   ├── user.service.test.ts          # Unit test (co-located)
│   ├── user.repository.ts
│   └── user.repository.test.ts
├── shared/
│   └── utils/
│       ├── validators.ts
│       └── validators.test.ts
tests/
├── integration/                      # Integration tests (real DB, real HTTP)
│   ├── users.integration.test.ts
│   └── auth.integration.test.ts
└── helpers/
    ├── test-app.ts                   # App factory for integration tests
    ├── test-db.ts                    # DB setup/teardown helpers
    └── fixtures/                     # Shared test data
        └── users.fixture.ts
```

- **Unit tests**: co-located with source as `*.test.ts` (or `*.spec.ts`).
- **Integration tests**: in `tests/integration/` with separate vitest config.
- **Test helpers/fixtures**: in `tests/helpers/`.

---

## Unit Tests — Vitest

### Structure (Arrange-Act-Assert)

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { UserService } from './user.service.js';
import type { UserRepository } from './user.repository.js';
import type { EmailService } from '../email/email.service.js';

describe('UserService', () => {
  let service: UserService;
  let mockRepo: UserRepository;
  let mockEmail: EmailService;

  beforeEach(() => {
    mockRepo = {
      findById: vi.fn(),
      findByEmail: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    };
    mockEmail = { sendWelcome: vi.fn() };

    service = new UserService(mockRepo, mockEmail, { error: vi.fn(), info: vi.fn(), warn: vi.fn() });
  });

  describe('create', () => {
    it('should create a user and send welcome email when input is valid', async () => {
      // Arrange
      const dto = { name: 'Alice', email: 'alice@example.com', role: 'viewer' as const };
      const createdUser = { id: '1', ...dto, createdAt: new Date() };
      vi.mocked(mockRepo.findByEmail).mockResolvedValue(null);
      vi.mocked(mockRepo.create).mockResolvedValue(createdUser);

      // Act
      const result = await service.create(dto);

      // Assert
      expect(result).toEqual(createdUser);
      expect(mockRepo.create).toHaveBeenCalledWith(dto);
      expect(mockEmail.sendWelcome).toHaveBeenCalledWith('alice@example.com', 'Alice');
    });

    it('should throw ConflictError when email already exists', async () => {
      // Arrange
      const dto = { name: 'Alice', email: 'existing@example.com', role: 'viewer' as const };
      vi.mocked(mockRepo.findByEmail).mockResolvedValue({ id: '2', ...dto, createdAt: new Date() });

      // Act & Assert
      await expect(service.create(dto)).rejects.toThrow(ConflictError);
      expect(mockRepo.create).not.toHaveBeenCalled();
    });
  });
});
```

### Naming Convention

```typescript
// Pattern: "should <expected behavior> when <condition>"
it('should return null when user does not exist', () => { /* ... */ });
it('should throw ValidationError when email format is invalid', () => { /* ... */ });
it('should return 401 when authorization header is missing', () => { /* ... */ });
```

### Mocking with `vi`

```typescript
import { vi, afterEach } from 'vitest';

// ✅ vi.fn() for function mocks
const mockLogger = { error: vi.fn(), warn: vi.fn(), info: vi.fn() };

// ✅ vi.spyOn for partial mocks
vi.spyOn(Date, 'now').mockReturnValue(1_710_000_000_000);

// ✅ vi.mock for module mocking
vi.mock('../email/email.service.js', () => ({
  EmailService: vi.fn().mockImplementation(() => ({
    sendWelcome: vi.fn().mockResolvedValue(undefined),
  })),
}));

// ✅ Restore mocks after each test to prevent leakage
afterEach(() => {
  vi.restoreAllMocks();
});

// ❌ Never mock what you don't own — write integration tests instead
// ❌ Never mock implementation details — mock interfaces/contracts
```

### Parameterized Tests

```typescript
import { it, expect } from 'vitest';

describe('validateEmail', () => {
  it.each([
    ['user@example.com', true],
    ['user@sub.domain.com', true],
    ['invalid', false],
    ['@missing-local.com', false],
    ['', false],
  ])('should return %s for input "%s"', (input, expected) => {
    expect(validateEmail(input)).toBe(expected);
  });
});

// Object syntax for complex cases
it.each([
  { age: -1, error: 'Age must be non-negative' },
  { age: 200, error: 'Age must be below 150' },
  { age: 25, error: null },
])('should validate age=$age → error=$error', ({ age, error }) => {
  const result = validateAge(age);
  if (error) {
    expect(result.errors).toContain(error);
  } else {
    expect(result.valid).toBe(true);
  }
});
```

---

## Integration Tests — Testcontainers

Use real infrastructure (PostgreSQL, Redis, etc.) via Testcontainers for integration tests. Never mock the database in integration tests.

```typescript
// tests/integration/users.integration.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { buildApp } from '../helpers/test-app.js';
import supertest from 'supertest';

describe('Users API', () => {
  let container: StartedPostgreSqlContainer;
  let request: ReturnType<typeof supertest>;

  beforeAll(async () => {
    container = await new PostgreSqlContainer('postgres:16-alpine').start();

    const app = await buildApp({
      DATABASE_URL: container.getConnectionUri(),
    });

    request = supertest(app.callback());
  }, 60_000); // allow time for container startup

  afterAll(async () => {
    await container.stop();
  });

  describe('POST /api/users', () => {
    it('should create a user and return 201', async () => {
      const response = await request
        .post('/api/users')
        .set('Authorization', `Bearer ${testAdminToken}`)
        .send({ name: 'Alice', email: 'alice@example.com', role: 'viewer' })
        .expect(201);

      expect(response.body).toMatchObject({
        id: expect.any(String),
        name: 'Alice',
        email: 'alice@example.com',
      });
    });

    it('should return 409 when email already exists', async () => {
      await createTestUser({ email: 'duplicate@example.com' });

      await request
        .post('/api/users')
        .set('Authorization', `Bearer ${testAdminToken}`)
        .send({ name: 'Bob', email: 'duplicate@example.com', role: 'viewer' })
        .expect(409);
    });
  });
});
```

---

## Security Tests (Mandatory)

Every HTTP endpoint must have security tests. These are integration tests, not unit tests.

```typescript
describe('POST /api/users — Security', () => {
  it('should return 401 when Authorization header is missing', async () => {
    await request.post('/api/users').send({ name: 'Alice', email: 'a@example.com' }).expect(401);
  });

  it('should return 403 when user lacks admin role', async () => {
    await request
      .post('/api/users')
      .set('Authorization', `Bearer ${viewerToken}`)
      .send({ name: 'Alice', email: 'a@example.com' })
      .expect(403);
  });

  it('should return 400 when request body fails validation', async () => {
    const response = await request
      .post('/api/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ name: 'A', email: 'not-an-email' }) // name too short, invalid email
      .expect(400);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });

  it('should return 400 and reject unknown properties (prevent mass assignment)', async () => {
    await request
      .post('/api/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ name: 'Alice', email: 'a@example.com', role: 'viewer', isAdmin: true }) // unknown field
      .expect(400); // or 201 with isAdmin silently stripped — document which behavior is expected
  });

  it('should not return password hash in response', async () => {
    const response = await request
      .post('/api/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ name: 'Alice', email: 'a@example.com', role: 'viewer', password: 'secret' })
      .expect(201);

    expect(response.body).not.toHaveProperty('passwordHash');
    expect(response.body).not.toHaveProperty('password');
  });

  it('should return 429 after exceeding rate limit', async () => {
    for (let i = 0; i < 11; i++) {
      await request.post('/api/auth/login').send({ email: 'test@example.com', password: 'wrong' });
    }
    await request
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'wrong' })
      .expect(429);
  });
});
```

---

## Test Data — Fixtures

```typescript
// tests/helpers/fixtures/users.fixture.ts
import type { User } from '../../src/users/user.model.js';

export function buildUser(overrides: Partial<User> = {}): User {
  return {
    id: 'test-user-id-1',
    name: 'Test User',
    email: 'test@example.com',
    role: 'viewer',
    createdAt: new Date('2024-01-01T00:00:00Z'),
    ...overrides,
  };
}

export const testUsers = {
  admin: buildUser({ id: 'admin-1', name: 'Admin', email: 'admin@example.com', role: 'admin' }),
  editor: buildUser({ id: 'editor-1', name: 'Editor', email: 'editor@example.com', role: 'editor' }),
  viewer: buildUser({ id: 'viewer-1', name: 'Viewer', email: 'viewer@example.com', role: 'viewer' }),
};
```

---

## Mocking Time

```typescript
import { vi, beforeEach, afterEach } from 'vitest';

describe('TokenService', () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date('2024-06-15T12:00:00Z'));
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('should mark token as expired after 15 minutes', () => {
    const token = createToken({ expiresInMs: 15 * 60 * 1000 });
    vi.advanceTimersByTime(16 * 60 * 1000);
    expect(isExpired(token)).toBe(true);
  });
});
```

---

## vitest.config for Integration Tests

```typescript
// vitest.integration.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['tests/integration/**/*.test.ts'],
    testTimeout: 30_000,  // containers need more time
    hookTimeout: 60_000,
    poolOptions: {
      forks: { singleFork: true }, // run integration tests sequentially
    },
  },
});
```

---

## package.json Test Scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:all": "npm run test && npm run test:integration"
  }
}
```

---

## Rules Summary

- **Co-locate** unit tests with source files.
- **Arrange-Act-Assert** in every test — never mix concerns.
- **Never mock the database** in integration tests — use Testcontainers.
- **Never use `Math.random()`** or real `Date.now()` in tests — mock them.
- **Never use implementation details** in assertions — test behavior and output.
- **Security tests are mandatory** — auth, validation, and rate limiting for every endpoint.
- **No `any` in test files** — type your mocks properly with `vi.Mocked<T>`.
- **Restore mocks** after each test with `afterEach(() => vi.restoreAllMocks())`.
