---
description: "Root instructions for JavaScript / TypeScript / Angular projects. Loaded automatically as background context in every interaction."
---

# Copilot Instructions — JavaScript / TypeScript / Angular

## Quick Reference

| Aspect | Standard |
|--------|----------|
| **Language** | TypeScript 5.x (strict mode) — JavaScript only for config files |
| **Runtime** | Node.js 20 LTS+ |
| **Frontend Framework** | Angular 17+ (standalone components, signals) |
| **Package Manager** | npm (with `package-lock.json`) or pnpm |
| **Linter** | ESLint 9+ (flat config) with `@typescript-eslint` and `@angular-eslint` |
| **Formatter** | Prettier |
| **Test Runner** | Jest (unit/integration) · Playwright (E2E) · Karma/Jest for Angular |
| **Build** | Angular CLI · Vite · esbuild |
| **CI/CD** | GitHub Actions |
| **Commit Style** | Conventional Commits (`feat`, `fix`, `docs`, `security`, …) |

## Architecture Principles

1. **TypeScript first** — All application code is TypeScript with `strict: true`. JavaScript is only acceptable for tooling configs (`eslint.config.mjs`, `prettier.config.js`).
2. **Standalone components** — Angular components, directives, and pipes use `standalone: true` (default since Angular 17). NgModules are legacy.
3. **Reactive patterns** — Prefer Angular Signals and RxJS operators over imperative state management.
4. **Separation of concerns** — Feature modules, shared libraries, and core services are clearly separated.
5. **Security by default** — Follow `security.instructions.md` in every code generation and review.

## File Organization

Follow the conventions in `project-structure.instructions.md` for directory layout, `tsconfig.json` configuration, and CI/CD pipeline setup.

## Coding Conventions

Follow `javascript-coding.instructions.md` for:

- Naming conventions (camelCase, PascalCase, UPPER_SNAKE_CASE).
- TypeScript strict typing and utility types.
- Angular component patterns (signals, inputs, outputs, dependency injection).
- Async patterns (RxJS, Promises, `async`/`await`).
- Error handling and logging.

## Documentation

Follow `documentation.instructions.md` for:

- TSDoc comments on public APIs.
- README structure and CHANGELOG format.
- Angular component documentation (inputs, outputs, usage examples).
- Architecture Decision Records (ADRs).

## Testing

Follow `testing.instructions.md` for:

- Jest unit test patterns and mocking.
- Angular TestBed and component harnesses.
- Playwright E2E test structure.
- Security test requirements.

## Security

Follow `security.instructions.md` for:

- OWASP Top 10 for web applications.
- Angular-specific security (XSS, template injection, `bypassSecurityTrust*`).
- Dependency auditing (`npm audit`, Dependabot).
- Authentication and authorization patterns.

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `security`.

Scope examples: `auth`, `dashboard`, `api`, `shared`, `core`.

## Specialized Files

| File | Purpose |
|------|---------|
| `javascript-coding.instructions.md` | TypeScript/JS coding conventions and Angular patterns |
| `documentation.instructions.md` | TSDoc, README, CHANGELOG, ADR standards |
| `testing.instructions.md` | Jest, Angular testing, Playwright, security tests |
| `project-structure.instructions.md` | Directory layout, configs, CI/CD, Docker |
| `security.instructions.md` | OWASP Top 10, Angular security, dependency auditing |
| `code-review.agent.md` | Automated code review agent |
| `code-modernization.agent.md` | Migration agent (AngularJS → Angular 17+, JS → TS) |
