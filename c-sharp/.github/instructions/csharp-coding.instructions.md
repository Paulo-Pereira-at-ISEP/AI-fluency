---
description: "C# coding conventions and best practices for .NET projects"
applyTo: "**/*.cs,**/*.csx"
---

# C# Coding Guidelines

## High-Level Principles

- Prefer clarity and intent-revealing code over cleverness.
- Write code that is strongly typed, testable, and easy to maintain.
- Keep changes small and focused: prefer incremental edits that compile and pass tests.
- Optimize for readability first; optimize for performance only when profiling justifies it.
- Target the latest stable .NET LTS version unless project constraints dictate otherwise.

## Style & Formatting

### General Rules

- Follow the [Microsoft C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions).
- Use **`.editorconfig`** for consistent formatting across the team.
- Use **`dotnet format`** for automated formatting.
- Enable **Roslyn analyzers** and **`StyleCop.Analyzers`** for enforced code quality.
- Use **file-scoped namespaces** (`namespace MyApp;`) in C# 10+.
- Prefer expression-bodied members for single-line methods/properties.
- Use `var` when the type is obvious from the right side; use explicit types otherwise.

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Classes / Structs / Records | `PascalCase` | `DataProcessor`, `UserSession` |
| Interfaces | `IPascalCase` | `IRepository`, `ILogger` |
| Methods | `PascalCase`, verb-first | `CalculateTotal()`, `SendMessage()` |
| Properties | `PascalCase` | `UserCount`, `MaxRetries` |
| Public fields | `PascalCase` | `DefaultTimeout` |
| Private fields | `_camelCase` | `_cache`, `_logger` |
| Local variables | `camelCase` | `itemCount`, `maxRetries` |
| Parameters | `camelCase` | `userName`, `retryCount` |
| Constants | `PascalCase` | `MaxRetries`, `DefaultTimeout` |
| Enums | `PascalCase` (singular) | `Color.Red`, `Status.Active` |
| Type parameters | `T` prefix + `PascalCase` | `T`, `TKey`, `TValue` |
| Async methods | `PascalCase` + `Async` suffix | `GetUserAsync()`, `SaveDataAsync()` |
| Namespaces | `PascalCase`, dot-separated | `MyApp.Core.Services` |

### File Organization

- One type per file (exceptions: tightly coupled small types, e.g., a record + its extensions).
- File name must match the primary type name (`UserService.cs` for `UserService`).
- Order members within a class:
  1. Constants and static fields
  2. Private fields
  3. Constructors
  4. Public properties
  5. Public methods
  6. Private/internal methods
  7. Nested types

### Using Directives

- Place `using` directives **outside** the namespace (top of file).
- Order: `System.*` → third-party → project namespaces.
- Remove unused `using` directives.
- Use **global usings** in `GlobalUsings.cs` for commonly used namespaces.

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;
```

## Nullable Reference Types

- **Always** enable nullable reference types (`#nullable enable` or project-level `<Nullable>enable</Nullable>`).
- Annotate all public APIs with proper nullability.
- Avoid the null-forgiving operator (`!`) except in test code or validated scenarios.
- Use `is not null` / `is null` instead of `!= null` / `== null`.
- Use null-conditional (`?.`) and null-coalescing (`??`, `??=`) operators.
- Use `required` modifier (C# 11+) for properties that must be set at initialization.

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;
    private readonly IUserRepository _repository;

    public UserService(ILogger<UserService> logger, IUserRepository repository)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    public async Task<User?> GetByIdAsync(string userId, CancellationToken ct = default)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(userId);
        return await _repository.FindByIdAsync(userId, ct);
    }
}
```

## Modern C# Features (C# 10–12)

Prefer modern language features for conciseness and safety:

| Feature | Since | Usage |
|---------|-------|-------|
| File-scoped namespaces | C# 10 | `namespace MyApp.Services;` |
| Global usings | C# 10 | `global using System.Linq;` |
| Record types | C# 9 | Immutable DTOs: `public record UserDto(string Name, string Email);` |
| Primary constructors | C# 12 | `public class Service(ILogger logger, IRepo repo)` |
| Pattern matching | C# 8+ | `if (obj is User { IsActive: true } user)` |
| Switch expressions | C# 8 | `var result = status switch { ... };` |
| Collection expressions | C# 12 | `List<int> items = [1, 2, 3];` |
| Raw string literals | C# 11 | `"""multiline"""` |
| Required members | C# 11 | `public required string Name { get; init; }` |
| `init` accessors | C# 9 | Immutable-after-init properties |
| Target-typed new | C# 9 | `UserDto dto = new("Alice", "a@b.com");` |

## Records, Classes & Data Structures

- Use **`record`** types for immutable data transfer objects (DTOs) and value objects.
- Use **`record struct`** for small, stack-allocated value types.
- Use **`class`** for entities with identity and mutable state.
- Prefer **composition over inheritance**.
- Implement `IEquatable<T>` for value-semantic types (or use records).
- Use `IReadOnlyList<T>`, `IReadOnlyDictionary<TK, TV>` for public API return types to prevent mutation.

```csharp
// Immutable DTO
public record UserDto(string Name, string Email, Role Role);

// Value object with validation
public record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        if (!value.Contains('@'))
            throw new ArgumentException("Invalid email format", nameof(value));
        Value = value.ToLowerInvariant();
    }
}

// Entity with identity
public class User
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public required string Name { get; set; }
    public required EmailAddress Email { get; init; }
    public Role Role { get; set; } = Role.Viewer;
}
```

## Dependency Injection

- Use the built-in `Microsoft.Extensions.DependencyInjection` container.
- Register services by interface, not by concrete type.
- Use constructor injection exclusively; avoid service locator pattern.
- Prefer `AddScoped` for request-scoped services, `AddSingleton` for stateless services, `AddTransient` for lightweight/stateless factories.
- Use the `Options` pattern (`IOptions<T>`, `IOptionsSnapshot<T>`) for configuration.

```csharp
// Registration
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.Configure<SmtpSettings>(builder.Configuration.GetSection("Smtp"));

// Consumption
public class UserService(IUserRepository repository, IOptions<SmtpSettings> smtpOptions)
{
    public async Task<User?> GetByIdAsync(string id, CancellationToken ct) =>
        await repository.FindByIdAsync(id, ct);
}
```

## Async / Await

- Use `async`/`await` for all I/O-bound operations.
- **Always** accept and forward `CancellationToken` parameters.
- Suffix async methods with `Async`.
- Avoid `async void` except for event handlers.
- Avoid `.Result` and `.Wait()` — they can cause deadlocks.
- Use `Task.WhenAll` for independent parallel operations.
- Use `ValueTask<T>` for hot-path async methods that often complete synchronously.
- Prefer `await using` for async disposable resources.

```csharp
public async Task<IReadOnlyList<User>> GetActiveUsersAsync(CancellationToken ct = default)
{
    await using var connection = await _dbFactory.CreateConnectionAsync(ct);
    return await connection.QueryAsync<User>(
        "SELECT * FROM Users WHERE IsActive = @IsActive",
        new { IsActive = true },
        ct);
}
```

## Error Handling

- Catch **specific** exceptions; never catch bare `Exception` without re-throwing.
- Use guard clauses (`ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrWhiteSpace`) for parameter validation.
- Use custom exception classes inheriting from a project-level base exception.
- Use `when` clauses on catch blocks for conditional handling.
- Log exceptions with full context before re-throwing or wrapping.
- Use `ExceptionDispatchInfo.Capture` when re-throwing from a different context.

```csharp
public abstract class AppException : Exception
{
    public string Code { get; }
    protected AppException(string code, string message, Exception? inner = null)
        : base(message, inner) => Code = code;
}

public class NotFoundException : AppException
{
    public NotFoundException(string entity, object id)
        : base("NOT_FOUND", $"{entity} with ID '{id}' was not found.") { }
}

// Usage with guard clause and specific exception
public async Task<User> GetRequiredUserAsync(string userId, CancellationToken ct)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(userId);

    var user = await _repository.FindByIdAsync(userId, ct);
    return user ?? throw new NotFoundException(nameof(User), userId);
}
```

## Logging

- Use `Microsoft.Extensions.Logging` (`ILogger<T>`) — never `Console.WriteLine` in production code.
- Use **structured logging** with message templates (not string interpolation).
- Use **high-performance logging** with `LoggerMessage.Define` or `[LoggerMessage]` source generator for hot paths.
- Use appropriate log levels: `Trace` → `Debug` → `Information` → `Warning` → `Error` → `Critical`.
- Never log sensitive data (passwords, tokens, PII).

```csharp
public partial class UserService
{
    [LoggerMessage(Level = LogLevel.Information, Message = "User {UserId} retrieved successfully")]
    private static partial void LogUserRetrieved(ILogger logger, string userId);

    [LoggerMessage(Level = LogLevel.Warning, Message = "User {UserId} not found")]
    private static partial void LogUserNotFound(ILogger logger, string userId);
}
```

## LINQ

- Use LINQ method syntax for complex queries; use query syntax when it reads more naturally (joins, multiple `from`).
- Avoid LINQ in performance-critical loops; prefer `for`/`foreach` with manual optimization.
- Prefer `Any()` over `Count() > 0`.
- Use `FirstOrDefault` / `SingleOrDefault` with default value or null check.
- Materialize queries early (`.ToList()`, `.ToArray()`) to avoid multiple enumeration.

## Performance & Memory

- **Profile before optimizing.** Use BenchmarkDotNet for micro-benchmarks, `dotnet-trace` / `dotnet-counters` for runtime analysis.
- Use `Span<T>`, `ReadOnlySpan<T>`, and `Memory<T>` for high-performance buffer operations.
- Use **object pooling** (`ObjectPool<T>`, `ArrayPool<T>`) for frequently allocated objects.
- Use `StringBuilder` for string concatenation in loops.
- Use `IAsyncEnumerable<T>` for streaming large datasets.
- Prefer `struct` for small, frequently allocated value types (≤16 bytes, no reference fields).
- Avoid boxing: use generic constraints, `IEquatable<T>`, and typed collections.

## Security

> For comprehensive security guidelines, see [security.instructions.md](security.instructions.md).

- **Never** hardcode secrets, API keys, or credentials in source code — use User Secrets, environment variables, or Azure Key Vault.
- **Validate and sanitize all external inputs** using Data Annotations, FluentValidation, or manual checks.
- **Always** use parameterized queries — never string interpolation/concatenation for SQL.
- Prevent path traversal: validate and canonicalize file paths.
- Use `System.Security.Cryptography.RandomNumberGenerator` for cryptographic randomness — never `System.Random`.
- Set security headers (CSP, HSTS, X-Content-Type-Options) on all HTTP responses.
- Keep NuGet packages up to date; run `dotnet list package --vulnerable` regularly.
- Implement rate limiting on public-facing endpoints.
- Log security events (auth failures, input validation errors) but **never** log passwords, tokens, or PII.

## Notes for LLM Agent Behavior

- Prioritize suggestions that align with the existing code style and project conventions.
- Propose minimal, test-first edits. Prefer suggestions that are reversible and safe.
- When suggesting API changes, include migration steps and update all callers.
- Always include nullable annotations in suggested code.
- Generate code targeting the project's minimum .NET / C# version.
- Do not suggest running the application unless explicitly asked; provide code changes for the developer to test.
- **Never** generate code that uses string interpolation/concatenation for SQL queries.
- **Always** use parameterized queries when generating database code.
- **Always** use `RandomNumberGenerator` instead of `System.Random` for security-sensitive values.
- When generating API endpoints, include input validation, authentication, and error handling that does not leak internals.
- Flag any hardcoded credentials found in code and recommend User Secrets, environment variables, or a secrets manager.
