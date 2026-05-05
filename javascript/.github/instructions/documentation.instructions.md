---
description: "Documentation standards for JavaScript, TypeScript, and Angular projects."
---

# Documentation Standards — JavaScript / TypeScript / Angular

## General Principles

- Documentation is **code** — it lives alongside the source, is versioned, and is reviewed in PRs.
- Write documentation for **two audiences**: human developers and LLM agents.
- Prefer **self-documenting code** (expressive names, small functions) complemented by documentation that explains **why**, not **what**.
- Keep documentation **close to the code** it describes — inline TSDoc over external wikis.

## TSDoc Comments

Use [TSDoc](https://tsdoc.org/) for all public APIs. TSDoc is the standard for TypeScript documentation comments.

### Functions & Methods

```typescript
/**
 * Calculates the total price for a list of cart items, applying
 * any active discounts.
 *
 * @param items - The cart items to calculate the total for.
 * @param discount - Optional discount percentage (0-100).
 * @returns The calculated total price in cents.
 * @throws {InvalidArgumentError} If discount is outside the 0-100 range.
 *
 * @example
 * ```typescript
 * const total = calculateTotal(cartItems, 10);
 * console.log(total); // 9000 (cents)
 * ```
 */
export function calculateTotal(items: CartItem[], discount?: number): number {
  // ...
}
```

### Classes & Interfaces

```typescript
/**
 * Service responsible for user authentication and session management.
 *
 * Handles login, logout, token refresh, and permission checks.
 * Uses JWT tokens stored in HTTP-only cookies.
 *
 * @example
 * ```typescript
 * const authService = inject(AuthService);
 * const user = await firstValueFrom(authService.currentUser$);
 * ```
 */
@Injectable({ providedIn: 'root' })
export class AuthService {
  /**
   * Observable of the currently authenticated user.
   * Emits `null` when no user is logged in.
   */
  readonly currentUser$: Observable<User | null>;

  /**
   * Attempts to authenticate a user with the provided credentials.
   *
   * @param credentials - The login credentials (email + password).
   * @returns Observable that emits the authenticated user on success.
   * @throws {AuthenticationError} If credentials are invalid.
   */
  login(credentials: LoginCredentials): Observable<User> { ... }
}
```

### Interfaces & Types

```typescript
/**
 * Represents a product in the catalog.
 *
 * @remarks
 * Prices are stored in cents to avoid floating-point precision issues.
 * Use {@link formatPrice} to convert to display format.
 */
export interface Product {
  /** Unique product identifier (UUID v4). */
  readonly id: string;

  /** Human-readable product name (1-200 characters). */
  name: string;

  /** Price in cents. Must be non-negative. */
  priceInCents: number;

  /** Product categories for filtering. At least one required. */
  categories: string[];

  /** ISO 8601 timestamp of when the product was created. */
  readonly createdAt: string;
}
```

### Angular Components

```typescript
/**
 * Displays a user profile card with avatar, name, and role badge.
 *
 * @remarks
 * Uses `OnPush` change detection. All inputs are signal-based.
 *
 * @usageNotes
 * ```html
 * <app-user-card
 *   [user]="selectedUser()"
 *   [showActions]="true"
 *   (userSelected)="onUserSelect($event)"
 * />
 * ```
 *
 * @see {@link UserService} for fetching user data.
 */
@Component({
  selector: 'app-user-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
export class UserCardComponent {
  /** The user to display. Required. */
  readonly user = input.required<User>();

  /** Whether to show action buttons (edit, delete). Defaults to `false`. */
  readonly showActions = input(false);

  /** Emits when the user clicks the card. Payload is the selected user. */
  readonly userSelected = output<User>();
}
```

### TSDoc Tags Reference

| Tag | Usage |
|-----|-------|
| `@param name - desc` | Document a function parameter |
| `@returns` | Document the return value |
| `@throws {ErrorType}` | Document thrown errors |
| `@example` | Provide usage example (in code fence) |
| `@remarks` | Extended description or caveats |
| `@see` | Link to related code or docs |
| `@deprecated` | Mark as deprecated with migration path |
| `@defaultValue` | Document default value of optional param |
| `@usageNotes` | Angular-style usage notes with template examples |
| `@internal` | Mark as internal (not part of public API) |

## Inline Comments

```typescript
// ✅ GOOD: explains WHY
// We retry 3 times because the payment gateway has intermittent 503s
// during their deployment window (daily 02:00-02:15 UTC).
const MAX_RETRIES = 3;

// ✅ GOOD: warns about non-obvious behavior
// IMPORTANT: This array is sorted in-place by the API response.
// Clone before modifying if you need the original order.
const users = response.data;

// ❌ BAD: restates the code
// Set the name to the user's name
this.name = user.name;

// ✅ GOOD: TODO with context
// TODO(#1234): Replace with server-side pagination once the API supports it.
const allUsers = await this.loadAllUsers();
```

## README.md

Every project and significant library must have a README with the following structure:

```markdown
# Project Name

Brief description of what the project does (1-2 sentences).

## Prerequisites

- Node.js 20+
- npm 10+ or pnpm 9+
- Angular CLI 17+

## Getting Started

### Installation

npm install

### Development Server

ng serve

### Running Tests

npm test           # Unit tests
npm run test:e2e   # E2E tests

## Architecture

Brief overview of the architecture with reference to ADRs.

### Directory Structure

src/
├── app/
│   ├── core/          # Singleton services, guards, interceptors
│   ├── features/      # Feature modules (lazy-loaded)
│   ├── shared/        # Shared components, directives, pipes
│   └── app.config.ts  # Application configuration
├── assets/
├── environments/
└── styles/

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `API_URL` | Backend API base URL | `http://localhost:3000` |

## Deployment

How to build and deploy the application.

## Contributing

Link to contributing guidelines and code review process.
```

## CHANGELOG.md

Follow [Keep a Changelog](https://keepachangelog.com/) format:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- User profile page with avatar upload (#234).

### Fixed
- Dashboard chart not rendering on Safari (#256).

### Security
- Updated `express` to 4.19.2 to fix CVE-2024-XXXX.

## [1.2.0] - 2026-03-01

### Added
- Role-based access control for admin panel.

### Changed
- Migrated from `*ngIf` to `@if` control flow syntax.
```

## Architecture Decision Records (ADRs)

Store in `docs/adr/` with sequential numbering:

```
docs/
└── adr/
    ├── 0001-use-angular-standalone-components.md
    ├── 0002-choose-jest-over-karma.md
    ├── 0003-adopt-signal-based-state-management.md
    └── template.md
```

### ADR Template

```markdown
# ADR-NNNN: Title

## Status

Proposed | Accepted | Deprecated | Superseded by ADR-XXXX

## Context

What is the issue or decision that needs to be made?

## Decision

What was decided and why?

## Consequences

### Positive
- ...

### Negative
- ...

### Neutral
- ...

## Alternatives Considered

| Alternative | Pros | Cons | Reason for rejection |
|-------------|------|------|---------------------|
| ... | ... | ... | ... |
```

## API Documentation

### REST APIs (Node.js/Express/NestJS)

Use OpenAPI/Swagger with decorators or JSDoc annotations:

```typescript
/**
 * @openapi
 * /api/users/{id}:
 *   get:
 *     summary: Get user by ID
 *     tags: [Users]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *           format: uuid
 *     responses:
 *       200:
 *         description: User found
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/User'
 *       404:
 *         description: User not found
 */
router.get('/api/users/:id', getUserById);
```

### Angular Services (Compodoc)

Use [Compodoc](https://compodoc.app/) for Angular project documentation:

```bash
# Generate documentation
npx compodoc -p tsconfig.json -s

# Output: documentation/ directory with HTML docs
```

## Environment Documentation

Document all environment variables in a `.env.example` file:

```bash
# .env.example — Copy to .env and fill in values
# NEVER commit .env to version control

# API Configuration
API_URL=http://localhost:3000
API_TIMEOUT=5000

# Authentication
AUTH_ISSUER=https://auth.example.com
AUTH_AUDIENCE=my-app

# Feature Flags
ENABLE_DARK_MODE=true
ENABLE_ANALYTICS=false
```

## LLM Agent Directives

When generating or modifying documentation, the agent MUST:

- Add TSDoc comments to every public class, interface, type, function, and Angular component.
- Include `@param`, `@returns`, and `@throws` tags on every function.
- Add `@example` blocks with working code for complex APIs.
- Document Angular component inputs, outputs, and usage in templates.
- Use `@deprecated` with a migration path when marking legacy code.
- Keep inline comments focused on **why**, never on **what**.
- Update CHANGELOG.md entries categorized under the correct section.
- Ensure README.md stays in sync with actual project structure.
