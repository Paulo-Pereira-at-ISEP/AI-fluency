---
description: "Root instructions for TypeScript projects (Node.js, REST APIs, libraries, CLI tools). Loaded automatically as background context in every interaction."
---

# Copilot Instructions — TypeScript

## Quick Reference

| Aspect | Standard |
|--------|----------|
| **Language** | TypeScript 5.x (strict mode) — JavaScript only for config files |
| **Runtime** | Node.js 20 LTS+ |
| **Framework** | NestJS · Express · Fastify (API) · tsup · tsc (libraries) |
| **Package Manager** | npm (with `package-lock.json`) or pnpm |
| **Linter** | ESLint 9+ (flat config) with `typescript-eslint` |
| **Formatter** | Prettier |
| **Test Runner** | Vitest (preferred) · Jest |
| **Build** | tsup · tsc · esbuild |
| **Validation** | Zod |
| **CI/CD** | GitHub Actions |
| **Commit Style** | Conventional Commits (`feat`, `fix`, `docs`, `security`, …) |

## Architecture Principles

1. **TypeScript strict mode** — All code uses `strict: true` plus additional strictness flags. JavaScript is only acceptable for tooling configs.
2. **Type safety at boundaries** — All external inputs (HTTP, environment variables, config files, CLI args) are validated and typed with Zod.
3. **Immutability by default** — `const`, `Readonly<T>`, `as const`; mutation is the exception.
4. **Pure functions** — Prefer functional, side-effect-free logic; isolate side effects at the edges.
5. **Separation of concerns** — Domain logic, infrastructure (DB, HTTP), and application orchestration are clearly separated.
6. **Security by default** — Follow `security.instructions.md` in every code generation and review.
7. **Explicit over implicit** — Explicit return types, explicit `async`/`await`, explicit error handling.

## File Organization

Follow the conventions in `project-structure.instructions.md` for directory layout, `tsconfig.json` configuration, and CI/CD pipeline setup.

## Coding Conventions

Follow `typescript-coding.instructions.md` for:

- Naming conventions (camelCase, PascalCase, UPPER_SNAKE_CASE).
- TypeScript strict typing, generics, and advanced type patterns.
- Async patterns (`async`/`await`, error handling with `Result` types).
- Module system (ESM by default, CJS interop when required).
- Decorator patterns (NestJS and experimental decorators).
- Declaration files (`.d.ts`) for published libraries.

## Documentation

Follow `documentation.instructions.md` for:

- TSDoc comments on public APIs.
- README structure and CHANGELOG format.
- API documentation (OpenAPI/Swagger for REST APIs).
- Architecture Decision Records (ADRs).

## Testing

Follow `testing.instructions.md` for:

- Vitest unit test patterns and mocking.
- Integration test patterns with real dependencies (testcontainers).
- Security test requirements.
- Coverage thresholds.

## Security

Follow `security.instructions.md` for:

- OWASP Top 10 for APIs and Node.js backends.
- Input validation with Zod at every boundary.
- Authentication (JWT, OAuth2) and authorization (RBAC).
- Dependency auditing (`npm audit`, Dependabot).
- Secrets management (environment variables, vaults).

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `security`.

Scope examples: `auth`, `users`, `api`, `db`, `config`, `lib`.

## Specialized Files

| File | Purpose |
|------|---------|
| `typescript-coding.instructions.md` | TypeScript coding conventions, type system, async patterns |
| `documentation.instructions.md` | TSDoc, README, CHANGELOG, OpenAPI standards |
| `testing.instructions.md` | Vitest, integration tests, security tests |
| `project-structure.instructions.md` | Directory layout, tsconfig, ESLint, CI/CD |
| `security.instructions.md` | OWASP Top 10 for APIs, Zod validation, secrets management |
| `code-review.agent.md` | Automated code review agent |
| `code-modernization.agent.md` | Migration agent (JS → TS, CommonJS → ESM, legacy patterns → modern TS) |
