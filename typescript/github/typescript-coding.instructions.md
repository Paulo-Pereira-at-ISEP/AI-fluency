---
description: "Coding conventions and best practices for TypeScript projects: Node.js APIs, libraries, CLI tools, and NestJS applications."
---

# TypeScript — Coding Conventions

## General Principles

- **TypeScript strict mode is mandatory** — enable `strict: true` plus additional flags in every `tsconfig.json`.
- **No `any`** — use `unknown` when the type cannot be determined, then narrow with type guards.
- **Prefer immutability** — `const` by default, `let` only when reassignment is necessary, never `var`.
- **Explicit return types** on all exported functions and class methods.
- **Fail fast at boundaries** — validate all external input (HTTP, env vars, CLI args, files) with Zod before use.
- **Prefer pure functions** — isolate side effects (I/O, DB, HTTP) at the edges of the application.
- **Small, focused functions** — a function should do one thing; aim for ≤ 30 lines.

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Variables, functions | `camelCase` | `userName`, `getUserById()` |
| Classes, interfaces, types, enums | `PascalCase` | `UserService`, `HttpError` |
| Constants (true constants) | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT_MS` |
| Enum members | `PascalCase` | `HttpMethod.Get`, `UserRole.Admin` |
| Interfaces | `PascalCase`, **no** `I` prefix | `User`, `ApiResponse<T>` (not `IUser`) |
| Type aliases | `PascalCase` | `UserId`, `Nullable<T>` |
| Zod schemas | `PascalCase` + `Schema` suffix | `CreateUserSchema`, `EnvSchema` |
| File names | `kebab-case` | `user.service.ts`, `create-user.dto.ts` |
| Test files | `*.test.ts` or `*.spec.ts` | `user.service.test.ts` |
| Index files | `index.ts` (barrel exports) | `src/users/index.ts` |
| Declaration files | `*.d.ts` | `globals.d.ts` |
| Private class members | No prefix (use `private` keyword) | `private readonly repo: UserRepository` |

---

## TypeScript Strict Configuration

### tsconfig.json — Mandatory Settings

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedSideEffectImports": true
  }
}
```

---

## Type System — Core Patterns

### Explicit Annotations

```typescript
// ✅ Explicit return types on all public/exported functions
export function calculateTotal(items: CartItem[], discount = 0): number {
  return items.reduce((sum, item) => sum + item.priceInCents * item.quantity, 0) * (1 - discount);
}

// ✅ Explicit generic constraints
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

// ❌ Never use `any`
function process(data: any) { /* ... */ }

// ✅ Use `unknown` + type narrowing
function process(data: unknown): ProcessedData {
  if (!isRawData(data)) throw new TypeError('Invalid input');
  return transform(data);
}
```

### Utility Types

```typescript
// Prefer built-in utility types over manual equivalents
type UserUpdate    = Partial<User>;
type RequiredUser  = Required<User>;
type ReadonlyUser  = Readonly<User>;
type UserName      = Pick<User, 'firstName' | 'lastName'>;
type PublicUser    = Omit<User, 'passwordHash' | 'salt'>;

// Generic response wrapper
type ApiResponse<T> = {
  readonly data: T;
  readonly status: number;
  readonly message: string;
};

// Paginated result
type PaginatedResult<T> = {
  readonly items: T[];
  readonly total: number;
  readonly page: number;
  readonly pageSize: number;
};
```

### Discriminated Unions

```typescript
// ✅ Discriminated union for async state
type AsyncState<T> =
  | { readonly status: 'idle' }
  | { readonly status: 'loading' }
  | { readonly status: 'success'; readonly data: T }
  | { readonly status: 'error'; readonly error: Error };

// ✅ Result type for explicit error handling (no exceptions at domain layer)
type Result<T, E = Error> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

function ok<T>(value: T): Result<T> {
  return { ok: true, value };
}

function err<E = Error>(error: E): Result<never, E> {
  return { ok: false, error };
}
```

### Type Guards

```typescript
// ✅ Custom type guard with `is` predicate
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value && typeof (value as Record<string, unknown>).id === 'string' &&
    'email' in value && typeof (value as Record<string, unknown>).email === 'string'
  );
}

// ✅ Assertion function
function assertDefined<T>(value: T | null | undefined, name: string): asserts value is T {
  if (value == null) throw new Error(`Expected ${name} to be defined`);
}

// ✅ `satisfies` for type validation without widening
const httpConfig = {
  baseUrl: 'https://api.example.com',
  timeoutMs: 5000,
  retries: 3,
} satisfies HttpClientConfig;
```

### Advanced Types

```typescript
// ✅ Template literal types for type-safe event names / route paths
type EventName = `on${Capitalize<string>}`;
type ApiRoute = `/api/${string}`;

// ✅ Conditional types for library utilities
type Awaited<T> = T extends Promise<infer U> ? U : T;
type NonNullable<T> = T extends null | undefined ? never : T;

// ✅ Mapped types for transformations
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

type Optional<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

// ✅ `infer` for extracting nested types
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type ReturnTypeOf<F extends (...args: unknown[]) => unknown> = F extends (...args: unknown[]) => infer R ? R : never;
```

### Enums vs Union Types

```typescript
// ✅ Prefer string literal unions for simple discriminants
type UserRole = 'admin' | 'editor' | 'viewer';
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';

// ✅ `as const` objects for enum-like lookups with runtime access
const UserRole = {
  Admin: 'admin',
  Editor: 'editor',
  Viewer: 'viewer',
} as const;
type UserRole = (typeof UserRole)[keyof typeof UserRole];

// ✅ `const enum` only for numeric values erased at compile time
const enum HttpStatus {
  Ok = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  Forbidden = 403,
  NotFound = 404,
  InternalServerError = 500,
}

// ❌ Avoid regular `enum` — they generate runtime code and have surprising semantics
enum Direction { Up, Down } // ❌
```

---

## Runtime Validation — Zod

All external data (HTTP request bodies, query params, env vars, config files, CLI args) **must** be validated with Zod before use.

```typescript
import { z } from 'zod';

// ✅ Define schema once, derive TypeScript type from it
const CreateUserSchema = z.object({
  name: z.string().min(2).max(100).trim(),
  email: z.string().email().toLowerCase(),
  age: z.number().int().min(0).max(150).optional(),
  role: z.enum(['admin', 'editor', 'viewer']).default('viewer'),
});

type CreateUserDto = z.infer<typeof CreateUserSchema>;

// ✅ Parse at the boundary (HTTP handler, CLI entry, file reader)
function parseCreateUser(body: unknown): CreateUserDto {
  const result = CreateUserSchema.safeParse(body);
  if (!result.success) {
    throw new ValidationError('Invalid user input', result.error.flatten());
  }
  return result.data;
}

// ✅ Environment variable validation
const EnvSchema = z.object({
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
});

export const env = EnvSchema.parse(process.env);
// env.PORT is typed as `number`, not `string | undefined`
```

---

## Async Patterns

### `async`/`await` (preferred)

```typescript
// ✅ Always use async/await over raw Promises
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new HttpError(response.status, `User ${id} not found`);
  }
  return response.json() as Promise<User>;
}

// ✅ Parallel execution when order doesn't matter
async function loadDashboard(userId: string): Promise<Dashboard> {
  const [user, orders, notifications] = await Promise.all([
    fetchUser(userId),
    fetchOrders(userId),
    fetchNotifications(userId),
  ]);
  return { user, orders, notifications };
}

// ✅ Handle partial failures
async function loadWithFallback(ids: string[]): Promise<SettledResult[]> {
  const results = await Promise.allSettled(ids.map(id => fetchUser(id)));
  return results.map((result, i) => ({
    id: ids[i]!,
    user: result.status === 'fulfilled' ? result.value : null,
    error: result.status === 'rejected' ? result.reason : null,
  }));
}
```

### Error Handling

```typescript
// ✅ Domain errors as typed classes
export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id '${id}' not found`, 'NOT_FOUND', 404);
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly details: unknown,
  ) {
    super(message, 'VALIDATION_ERROR', 400);
  }
}

// ✅ Centralized error handler (Express example)
function errorHandler(err: unknown, req: Request, res: Response, _next: NextFunction): void {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: { code: err.code, message: err.message },
    });
    return;
  }
  // Unknown error — log full details internally, return generic response
  logger.error('Unhandled error', { err });
  res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: 'An unexpected error occurred' } });
}
```

### Result Pattern (no exceptions for expected failures)

```typescript
// ✅ Use Result<T, E> at domain/service layer to make failures explicit
async function findUser(id: string): Promise<Result<User, NotFoundError>> {
  const user = await userRepository.findById(id);
  if (!user) return err(new NotFoundError('User', id));
  return ok(user);
}

// Caller
const result = await findUser(userId);
if (!result.ok) {
  logger.warn('User not found', { userId });
  return res.status(404).json({ error: result.error.message });
}
const { value: user } = result;
```

---

## Module System

### ESM (default for all new projects)

```typescript
// ✅ Use ES module imports/exports
import { createUser } from './user.service.js';    // .js extension required in ESM
export { UserService } from './user.service.js';
export type { User, CreateUserDto } from './user.model.js';

// ✅ package.json for ESM library
// { "type": "module", "exports": { ".": { "import": "./dist/index.js", "types": "./dist/index.d.ts" } } }
```

### Barrel Exports (index.ts)

```typescript
// src/users/index.ts — export only the public API surface
export { UserService } from './user.service.js';
export type { User, CreateUserDto, UpdateUserDto } from './user.model.js';
// Do NOT export internal implementations, repositories, mappers
```

---

## Classes and Dependency Injection

```typescript
// ✅ Use constructor injection for testability
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailService: EmailService,
    private readonly logger: Logger,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) throw new ConflictError('User', 'email', dto.email);

    const user = await this.userRepository.create(dto);
    await this.emailService.sendWelcome(user.email, user.name);
    this.logger.info('User created', { userId: user.id });
    return user;
  }
}

// ✅ Use interfaces for dependencies (supports mocking in tests)
export interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(dto: CreateUserDto): Promise<User>;
  update(id: string, dto: UpdateUserDto): Promise<User>;
  delete(id: string): Promise<void>;
}
```

---

## NestJS Patterns (when using NestJS)

```typescript
// ✅ Use class-validator + class-transformer for DTO validation
import { IsEmail, IsString, MinLength, MaxLength } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @IsEmail()
  @Transform(({ value }) => (value as string).toLowerCase())
  email: string;
}

// ✅ Use Zod pipes for Zod-based validation
import { ZodValidationPipe } from 'nestjs-zod';
app.useGlobalPipes(new ZodValidationPipe());

// ✅ Typed controllers
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponseDto> {
    return this.usersService.findOne(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    return this.usersService.create(dto);
  }
}
```

---

## Code Style Rules

- **Max line length**: 100 characters (Prettier enforced).
- **No unused variables or imports** — ESLint `@typescript-eslint/no-unused-vars` at `error`.
- **No floating promises** — ESLint `@typescript-eslint/no-floating-promises` at `error`.
- **No misused promises** — ESLint `@typescript-eslint/no-misused-promises` at `error`.
- **No non-null assertions** (`!`) except at tested boundaries.
- **Prefer `??` over `||`** for nullish coalescing (avoids falsy pitfalls).
- **Prefer `?.` optional chaining** over manual null checks.
- **Import ordering**: Node built-ins → external packages → internal modules (ESLint enforced).
