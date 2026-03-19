---
description: "Documentation standards for Java projects with Spring Boot."
---

# Documentation Standards — Java

## General Principles

- Documentation is **code** — it lives alongside the source, is versioned, and is reviewed in PRs.
- Write documentation for **two audiences**: human developers and LLM agents.
- Prefer **self-documenting code** (expressive names, small methods) complemented by documentation that explains **why**, not **what**.
- Keep documentation **close to the code** it describes — Javadoc over external wikis.

## Javadoc Comments

Use Javadoc for all public and protected APIs. Follow the [Oracle Javadoc guidelines](https://www.oracle.com/technical-resources/articles/java/javadoc-tool.html).

### Classes & Interfaces

```java
/**
 * Service responsible for user lifecycle management.
 *
 * <p>Handles user creation, updates, deactivation, and role management.
 * All operations are transactional and publish domain events via
 * {@link ApplicationEventPublisher}.
 *
 * <p>This service validates input using Jakarta Bean Validation and
 * delegates persistence to {@link UserRepository}.
 *
 * @author Team Backend
 * @since 1.0.0
 * @see UserRepository
 * @see UserDto
 */
@Service
public class UserService { ... }
```

### Methods

```java
/**
 * Creates a new user with the specified details.
 *
 * <p>The password is hashed using bcrypt before storage. A
 * {@link UserCreatedEvent} is published after successful creation.
 *
 * @param request the user creation request, validated with {@link Valid}
 * @return the created user as a DTO
 * @throws DuplicateEmailException if a user with the same email already exists
 * @throws IllegalArgumentException if the request contains invalid data
 *
 * @since 1.0.0
 */
@Transactional
public UserDto create(@Valid CreateUserRequest request) { ... }
```

### Records & DTOs

```java
/**
 * Represents a user response returned by the API.
 *
 * <p>Prices are in cents to avoid floating-point precision issues.
 * Timestamps follow ISO 8601 format.
 *
 * @param id        unique user identifier
 * @param name      full display name (2–100 characters)
 * @param email     email address (unique, lowercase)
 * @param role      assigned role determining access permissions
 * @param createdAt ISO 8601 timestamp of account creation
 */
public record UserDto(
    Long id,
    String name,
    String email,
    UserRole role,
    Instant createdAt
) {}
```

### Enums

```java
/**
 * Represents the possible states of an order in its lifecycle.
 *
 * <p>State transitions:
 * <pre>
 * PENDING → CONFIRMED → SHIPPED → DELIVERED
 *                   ↘ CANCELLED
 * </pre>
 */
public enum OrderStatus {
    /** Order created but not yet confirmed by payment. */
    PENDING,

    /** Payment confirmed; awaiting shipment. */
    CONFIRMED,

    /** Order shipped; tracking number assigned. */
    SHIPPED,

    /** Order delivered to the customer. */
    DELIVERED,

    /** Order cancelled (before shipment only). */
    CANCELLED
}
```

### Javadoc Tags Reference

| Tag | Usage |
|-----|-------|
| `@param name description` | Document a method or constructor parameter |
| `@return description` | Document the return value |
| `@throws ExceptionType description` | Document thrown exceptions |
| `@see ClassName#method` | Link to related code |
| `@since version` | Version when the element was introduced |
| `@deprecated explanation` | Mark as deprecated with migration path |
| `@author name` | Author attribution (class-level) |
| `{@link ClassName}` | Inline link to another type |
| `{@code expression}` | Inline code formatting |
| `<p>` | Paragraph separator in descriptions |
| `<pre>` | Preformatted text (diagrams, examples) |

## Inline Comments

```java
// ✅ GOOD: explains WHY
// Retry 3 times because the payment gateway has intermittent 503s
// during their deployment window (daily 02:00-02:15 UTC).
private static final int MAX_RETRIES = 3;

// ✅ GOOD: warns about non-obvious behavior
// IMPORTANT: This list is sorted in-place by the JPA query.
// Clone before modifying if you need the original order.
List<User> users = repository.findAllSorted();

// ❌ BAD: restates the code
// Set the name to the user's name
this.name = user.getName();

// ✅ GOOD: TODO with context
// TODO(#1234): Replace with server-side pagination once the API supports cursor-based paging.
List<User> allUsers = repository.findAll();
```

## README.md

Every project and significant module must have a README:

```markdown
# Project Name

Brief description of what the project does (1-2 sentences).

## Prerequisites

- Java 21 (recommend SDKMAN for version management)
- Maven 3.9+
- Docker & Docker Compose (for integration tests)

## Getting Started

### Build

./mvnw clean verify

### Run

./mvnw spring-boot:run

### Run Tests

./mvnw test                         # Unit tests
./mvnw verify -Pintegration-test    # Integration tests

## Architecture

Brief overview with reference to ADRs.

### Module Structure

my-app/
├── my-app-domain/       # Domain entities, value objects, ports
├── my-app-application/  # Use cases, input/output ports
├── my-app-infra/        # Repository implementations, external APIs
└── my-app-web/          # REST controllers, filters, security config

## Configuration

| Property | Description | Default |
|----------|-------------|---------|
| `app.security.jwt-secret` | JWT signing secret | (required) |
| `app.cache.ttl-seconds` | Cache TTL in seconds | `300` |

## API Documentation

Swagger UI: http://localhost:8080/swagger-ui.html

## Deployment

How to build and deploy the application.

## Contributing

Link to contributing guidelines and code review process.
```

## CHANGELOG.md

Follow [Keep a Changelog](https://keepachangelog.com/):

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- User profile endpoint with avatar upload (#234).

### Fixed
- Race condition in order processing (#256).

### Security
- Updated Spring Boot to 3.3.x to fix CVE-2024-XXXX.

## [1.2.0] - 2026-03-01

### Added
- Role-based access control for admin endpoints.

### Changed
- Migrated from Java 17 to Java 21 with virtual threads.
```

## Architecture Decision Records (ADRs)

Store in `docs/adr/` with sequential numbering:

```
docs/
└── adr/
    ├── 0001-use-spring-boot-3.md
    ├── 0002-choose-postgresql-over-mysql.md
    ├── 0003-adopt-hexagonal-architecture.md
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

## OpenAPI / Swagger Documentation

### Springdoc OpenAPI

```java
// ✅ Document endpoints with OpenAPI annotations
@Operation(
    summary = "Get user by ID",
    description = "Returns a single user by their unique identifier.",
    responses = {
        @ApiResponse(responseCode = "200", description = "User found",
            content = @Content(schema = @Schema(implementation = UserDto.class))),
        @ApiResponse(responseCode = "404", description = "User not found",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
    }
)
@GetMapping("/{id}")
public ResponseEntity<UserDto> getById(@PathVariable Long id) { ... }

// ✅ Document DTOs
@Schema(description = "Request to create a new user")
public record CreateUserRequest(
    @Schema(description = "User's full name", example = "Alice Smith",
            minLength = 2, maxLength = 100)
    @NotBlank @Size(min = 2, max = 100)
    String name,

    @Schema(description = "Email address (must be unique)", example = "alice@example.com")
    @NotBlank @Email
    String email
) {}
```

### application.yml

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operationsSorter: method
    tagsSorter: alpha
  default-produces-media-type: application/json
```

## Package-level Documentation

Use `package-info.java` for package-level Javadoc:

```java
/**
 * User management domain.
 *
 * <p>This package contains the core domain entities, value objects,
 * and repository interfaces for user management. It has no dependencies
 * on Spring or any infrastructure framework.
 *
 * <h2>Key Types</h2>
 * <ul>
 *   <li>{@link com.example.domain.user.User} — aggregate root</li>
 *   <li>{@link com.example.domain.user.Email} — validated email value object</li>
 *   <li>{@link com.example.domain.user.UserRepository} — persistence port</li>
 * </ul>
 */
package com.example.domain.user;
```

## LLM Agent Directives

When generating or modifying documentation, the agent MUST:

- Add Javadoc to every public class, interface, record, enum, and method.
- Include `@param`, `@return`, and `@throws` tags on every public method.
- Document record components using `@param` in the record Javadoc.
- Use `{@link}` for cross-references to other types.
- Add `@since` tags when introducing new public APIs.
- Use `@deprecated` with a migration path when deprecating code.
- Keep inline comments focused on **why**, never on **what**.
- Update CHANGELOG.md entries categorized under the correct section.
- Ensure README.md stays in sync with actual project structure.
- Add OpenAPI `@Operation` and `@Schema` annotations on all REST endpoints and DTOs.
