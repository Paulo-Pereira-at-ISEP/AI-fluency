---
description: "Documentation standards for TypeScript Node.js projects: TSDoc, README, CHANGELOG, OpenAPI, and Architecture Decision Records."
---

# Documentation Standards — TypeScript / Node.js

## General Principles

- Documentation is **code** — it lives alongside the source, is versioned, and is reviewed in PRs.
- Write documentation for **two audiences**: human developers and LLM agents.
- Prefer **self-documenting code** (expressive names, small functions) complemented by documentation that explains **why**, not **what**.
- Keep documentation **close to the code** it describes — inline TSDoc over external wikis.
- **Public API = documented API** — every exported function, class, interface, and type must have a TSDoc comment.

---

## TSDoc Comments

Use [TSDoc](https://tsdoc.org/) for all public APIs. TSDoc is the TypeScript documentation standard, compatible with VS Code IntelliSense and documentation generators (TypeDoc).

### Functions & Methods

```typescript
/**
 * Retrieves a user by their unique identifier.
 *
 * @param id - The user's UUID (v4 format).
 * @returns The user if found, or `null` if no user with that ID exists.
 * @throws {DatabaseError} If the database query fails.
 *
 * @example
 * ```typescript
 * const user = await userService.findById('a3e7d9f1-...');
 * if (!user) throw new NotFoundError('User', id);
 * ```
 */
export async function findById(id: string): Promise<User | null> {
  // ...
}
```

### Classes & Services

```typescript
/**
 * Service responsible for user lifecycle management.
 *
 * Handles creation, retrieval, update, and deletion of users.
 * All operations are validated with Zod before execution.
 * Passwords are hashed with bcrypt (cost factor 12) before storage.
 *
 * @example
 * ```typescript
 * const service = new UserService(userRepository, emailService, logger);
 * const user = await service.create({ name: 'Alice', email: 'alice@example.com' });
 * ```
 */
export class UserService {
  /**
   * Creates a new user and sends a welcome email.
   *
   * @param dto - Validated user creation data.
   * @returns The newly created user (without password hash).
   * @throws {ConflictError} If the email address is already registered.
   */
  async create(dto: CreateUserDto): Promise<User> { /* ... */ }
}
```

### Interfaces & Types

```typescript
/**
 * Represents a registered user in the system.
 *
 * @remarks
 * `passwordHash` is never returned in API responses — use {@link PublicUser} for that.
 * Prices are stored in cents to avoid floating-point precision issues.
 */
export interface User {
  /** Unique user identifier (UUID v4). */
  readonly id: string;

  /** User's display name (2-100 characters). */
  name: string;

  /** Normalized email address (lowercase). */
  email: string;

  /** User role controlling permissions. */
  role: UserRole;

  /** Bcrypt hash of the user's password. Never expose in API responses. */
  readonly passwordHash: string;

  /** ISO 8601 timestamp of account creation. */
  readonly createdAt: string;
}

/**
 * User data safe to return in API responses (no sensitive fields).
 */
export type PublicUser = Omit<User, 'passwordHash'>;
```

### Zod Schemas

```typescript
/**
 * Validation schema for user creation requests.
 *
 * @remarks
 * - `email` is normalized to lowercase.
 * - `name` is trimmed of leading/trailing whitespace.
 * - Unknown properties are rejected (`.strict()`).
 */
export const CreateUserSchema = z.object({
  name: z.string().min(2).max(100).trim(),
  email: z.string().email().toLowerCase(),
  role: z.enum(['admin', 'editor', 'viewer']).default('viewer'),
}).strict();
```

### TSDoc Tags Reference

| Tag | Usage |
|-----|-------|
| `@param name - desc` | Document a function parameter |
| `@returns` | Document the return value |
| `@throws {ErrorType}` | Document thrown errors |
| `@example` | Provide a usage example (in code fence) |
| `@remarks` | Extended description or important caveats |
| `@see` | Link to related code (`{@link OtherClass}`) or docs |
| `@deprecated` | Mark as deprecated with migration guidance |
| `@internal` | Mark as not part of the public API (excluded from docs) |
| `@alpha` / `@beta` | Mark stability level for library APIs |

---

## README Structure

Every repository must have a README covering:

```markdown
# Package Name

> One-line description of what the package does.

## Features

- Feature 1
- Feature 2

## Requirements

- Node.js 20+
- PostgreSQL 16+

## Installation

```bash
npm install my-package
```

## Quick Start

```typescript
import { createServer } from './src/app.js';
const app = await createServer();
app.listen(3000);
```

## Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | ✅ | — | PostgreSQL connection string |
| `JWT_SECRET` | ✅ | — | Secret key (min 32 chars) |
| `PORT` | ❌ | `3000` | HTTP server port |

Copy `.env.example` to `.env` and fill in the values.

## Development

```bash
npm install
cp .env.example .env
# edit .env
npm run dev
```

## Testing

```bash
npm test               # unit tests
npm run test:coverage  # with coverage report
npm run test:integration # integration tests (requires Docker)
```

## API Documentation

OpenAPI spec: [`docs/openapi.yaml`](docs/openapi.yaml)  
Interactive docs (dev): `http://localhost:3000/docs`

## Architecture

See [`docs/adr/`](docs/adr/) for Architecture Decision Records.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

MIT
```

---

## CHANGELOG (Keep a Changelog format)

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] - 2024-06-15

### Added
- Rate limiting on auth endpoints (10 req / 15 min).
- `PATCH /api/users/:id` endpoint for partial updates.

### Changed
- Zod validation now rejects unknown properties on all schemas.

### Fixed
- Token refresh no longer loops on 401 responses.

### Security
- Updated `jsonwebtoken` to 9.0.2 (CVE-2022-23529).

## [1.1.0] - 2024-05-01

### Added
- OpenAPI 3.1 specification at `docs/openapi.yaml`.
```

---

## OpenAPI / Swagger (REST APIs)

Maintain an `openapi.yaml` alongside the code. For NestJS, generate it from decorators. For Express/Fastify, maintain it manually or use `zod-to-openapi`.

```yaml
# docs/openapi.yaml
openapi: '3.1.0'
info:
  title: My API
  version: '1.0.0'
  description: REST API for the My App service.

servers:
  - url: 'https://api.example.com'
    description: Production
  - url: 'http://localhost:3000'
    description: Development

security:
  - bearerAuth: []

paths:
  /api/users/{id}:
    get:
      summary: Get user by ID
      operationId: getUserById
      tags: [Users]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      required: [id, name, email, role, createdAt]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
          minLength: 2
          maxLength: 100
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, editor, viewer]
        createdAt:
          type: string
          format: date-time

  responses:
    Unauthorized:
      description: Missing or invalid authentication token
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

---

## Architecture Decision Records (ADRs)

```markdown
<!-- docs/adr/0001-use-zod-for-validation.md -->
# ADR 0001: Use Zod for Runtime Validation

**Status**: Accepted  
**Date**: 2024-01-15  
**Author**: Team

## Context

We need runtime input validation for HTTP requests and environment variables.
Options considered: Joi, Yup, class-validator, Zod.

## Decision

Use Zod for all runtime validation.

## Rationale

- Native TypeScript integration — types are inferred from schemas, no duplication.
- Schema-first — one schema serves as validation, type definition, and documentation.
- `.strict()` prevents mass-assignment attacks by rejecting unknown properties.
- `z.infer<typeof Schema>` eliminates the risk of type/schema drift.

## Consequences

- All DTOs are Zod schemas, not classes (no `class-transformer` needed).
- NestJS projects use `nestjs-zod` instead of `class-validator`.
- OpenAPI specs can be generated from Zod schemas via `zod-to-openapi`.
```

---

## .env.example (committed to version control)

```dotenv
# Application
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mydb

# Security — generate with: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
JWT_SECRET=

# Optional — Redis for rate limiting and session cache
REDIS_URL=redis://localhost:6379

# External services
EMAIL_API_KEY=
ALLOWED_ORIGINS=http://localhost:4200
```
