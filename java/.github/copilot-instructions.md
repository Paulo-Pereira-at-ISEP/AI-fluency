---
description: "Root instructions for Java projects. Loaded automatically as background context in every interaction."
---

# Copilot Instructions — Java

## Quick Reference

| Aspect | Standard |
|--------|----------|
| **Language** | Java 21 LTS (with preview features where stable) |
| **Build Tool** | Maven 3.9+ (preferred) · Gradle 8+ (Kotlin DSL) |
| **Framework** | Spring Boot 3.x / Spring Framework 6.x |
| **Test Runner** | JUnit 5 · Mockito · AssertJ |
| **Linter / Static Analysis** | Checkstyle · SpotBugs · Error Prone · SonarQube |
| **Formatter** | google-java-format · Spotless |
| **Dependency Management** | Maven BOM / Gradle version catalogs |
| **CI/CD** | GitHub Actions |
| **Commit Style** | Conventional Commits (`feat`, `fix`, `docs`, `security`, …) |

## Architecture Principles

1. **Clean Architecture** — Domain logic has zero framework dependencies. Dependencies point inward.
2. **Dependency Injection** — Use Spring constructor injection (no field injection, no `@Autowired` on fields).
3. **Immutability by default** — Prefer records, `final` fields, and unmodifiable collections.
4. **Fail fast** — Validate inputs at boundaries; throw meaningful exceptions early.
5. **Security by default** — Follow `security.instructions.md` in every code generation and review.

## File Organization

Follow the conventions in `project-structure.instructions.md` for directory layout, Maven/Gradle configuration, and CI/CD pipeline setup.

## Coding Conventions

Follow `java-coding.instructions.md` for:

- Naming conventions (camelCase, PascalCase, UPPER_SNAKE_CASE).
- Modern Java features (records, sealed classes, pattern matching, virtual threads).
- Spring Boot patterns (DI, configuration, profiles).
- Error handling and logging.

## Documentation

Follow `documentation.instructions.md` for:

- Javadoc comments on public APIs.
- README structure and CHANGELOG format.
- OpenAPI/Swagger documentation.
- Architecture Decision Records (ADRs).

## Testing

Follow `testing.instructions.md` for:

- JUnit 5 test patterns and lifecycle.
- Mockito mocking and AssertJ assertions.
- Spring Boot integration tests with `@SpringBootTest`.
- Testcontainers for database and infrastructure tests.
- Security test requirements.

## Security

Follow `security.instructions.md` for:

- OWASP Top 10 for Java/Spring applications.
- Spring Security configuration and best practices.
- Input validation with Jakarta Bean Validation.
- Dependency auditing (OWASP Dependency-Check, Dependabot).

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `security`.

Scope examples: `auth`, `api`, `domain`, `infra`, `web`.

## Specialized Files

| File | Purpose |
|------|---------|
| `java-coding.instructions.md` | Java coding conventions and Spring patterns |
| `documentation.instructions.md` | Javadoc, OpenAPI, README, CHANGELOG, ADR standards |
| `testing.instructions.md` | JUnit 5, Mockito, AssertJ, Testcontainers, security tests |
| `project-structure.instructions.md` | Maven/Gradle layout, profiles, CI/CD, Docker |
| `security.instructions.md` | OWASP Top 10, Spring Security, dependency auditing |
| `code-review.agent.md` | Automated code review agent |
| `code-modernization.agent.md` | Migration agent (Java 8 → 21, Spring Boot 2 → 3) |
