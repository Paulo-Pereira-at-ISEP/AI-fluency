---
description: "Documentation standards and conventions for C# / .NET projects"
applyTo: "**/*.cs,**/*.csx,**/*.md,**/*.xml"
---

# Documentation Standards

## High-Level Principles

- Documentation is part of the codebase: it must be maintained, reviewed, and tested alongside code.
- Write documentation for two audiences: **developers** (code-level docs) and **users/consumers** (API docs, guides).
- Prefer self-documenting code (clear names, strong types, small methods) supplemented by documentation for intent and context.

## XML Documentation Comments

### General Rules

- All public types, methods, properties, and events **must** have XML documentation comments.
- Use `///` triple-slash comments (XML doc comments) as the default convention.
- Keep the `<summary>` concise (one sentence). Add `<remarks>` for extended explanations.
- Write in third person present tense ("Gets the user" not "Get the user").

### Standard XML Tags

| Tag | Usage |
|-----|-------|
| `<summary>` | Brief description of the member |
| `<param name="">` | Describe each parameter |
| `<returns>` | Describe the return value |
| `<exception cref="">` | Document exceptions that may be thrown |
| `<remarks>` | Additional details, usage notes, algorithm descriptions |
| `<example>` | Code examples |
| `<see cref=""/>` | Cross-reference to another type or member |
| `<seealso cref=""/>` | Related types or members |
| `<typeparam name="">` | Describe generic type parameters |
| `<value>` | Describe the value of a property |
| `<inheritdoc/>` | Inherit documentation from base type or interface |

### Method Documentation

```csharp
/// <summary>
/// Calculates the discounted price for a given item.
/// </summary>
/// <param name="price">The original price. Must be non-negative.</param>
/// <param name="discountRate">
/// The discount rate as a decimal (e.g., 0.15 for 15%).
/// Must be between 0.0 and 1.0.
/// </param>
/// <param name="minPrice">The minimum allowed price after discount.</param>
/// <returns>The discounted price, guaranteed to be ≥ <paramref name="minPrice"/>.</returns>
/// <exception cref="ArgumentOutOfRangeException">
/// Thrown when <paramref name="price"/> is negative or
/// <paramref name="discountRate"/> is outside the range [0, 1].
/// </exception>
/// <example>
/// <code>
/// var result = CalculateDiscount(100.0m, 0.2m);    // 80.0
/// var capped = CalculateDiscount(50.0m, 0.9m, 10m); // 10.0
/// </code>
/// </example>
public decimal CalculateDiscount(decimal price, decimal discountRate, decimal minPrice = 0m)
{
    ...
}
```

### Class Documentation

```csharp
/// <summary>
/// A thread-safe queue for managing background tasks.
/// </summary>
/// <remarks>
/// Provides methods to enqueue, dequeue, and inspect tasks.
/// Tasks are processed in FIFO order with optional priority support.
/// This class is safe for concurrent access from multiple threads.
/// </remarks>
/// <typeparam name="T">The type of tasks managed by the queue.</typeparam>
/// <example>
/// <code>
/// var queue = new TaskQueue&lt;ProcessingJob&gt;(maxSize: 100);
/// queue.Enqueue(new ProcessingJob("data"));
/// var job = await queue.DequeueAsync(ct);
/// </code>
/// </example>
public class TaskQueue<T> where T : class
{
    ...
}
```

### Interface Documentation

```csharp
/// <summary>
/// Defines the contract for user persistence operations.
/// </summary>
/// <remarks>
/// Implementations should be registered as scoped services in the DI container.
/// All methods accept a <see cref="CancellationToken"/> for cooperative cancellation.
/// </remarks>
public interface IUserRepository
{
    /// <summary>
    /// Finds a user by their unique identifier.
    /// </summary>
    /// <param name="id">The unique user identifier.</param>
    /// <param name="ct">Cancellation token.</param>
    /// <returns>The user if found; otherwise, <see langword="null"/>.</returns>
    Task<User?> FindByIdAsync(string id, CancellationToken ct = default);
}
```

## Inline Comments

- Use inline comments sparingly — only to explain **why**, not **what**.
- Place comments on their own line above the code they explain.
- Do not comment obvious code.
- Use `// TODO:`, `// FIXME:`, `// HACK:`, `// NOTE:` prefixes for actionable comments.

```csharp
// Good: explains intent
// Retry with exponential backoff because the upstream API
// is rate-limited and returns 429 during peak hours.
for (int attempt = 0; attempt < maxRetries; attempt++)
{
    ...
}

// Bad: restates the code
// Increment counter by 1
counter++;
```

## README Files

Every project and significant library should have a `README.md` containing:

1. **Title and description**: What the project/library does.
2. **Prerequisites**: Required SDK version, tools, and runtime.
3. **Quick start**: `dotnet restore` / `dotnet build` / `dotnet run` commands.
4. **Configuration**: Required environment variables/settings, `appsettings.json` structure.
5. **Usage**: Key features and API overview.
6. **Development**: How to run tests, format code, and contribute.
7. **Architecture**: High-level design (optional, for larger projects).

## Changelog

- Maintain a `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/) format.
- Group changes by: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.
- Link each version to its Git tag or comparison URL.

```markdown
## [1.2.0] - 2026-03-15

### Added
- Support for batch processing in `DataPipeline`.
- New `--dry-run` flag for CLI commands.

### Fixed
- Race condition in `TaskQueue.DequeueAsync()` under high concurrency.

### Security
- Updated `System.Text.Json` to address CVE-2026-XXXX.

### Deprecated
- `ProcessSingle()` — use `ProcessBatchAsync()` with a single-item list instead.
```

## API Documentation

- Use **DocFX** or **Swagger / Swashbuckle** (for Web APIs) to generate documentation from XML doc comments.
- Enable `<GenerateDocumentationFile>true</GenerateDocumentationFile>` in `.csproj` files.
- Treat XML doc warnings (`CS1591`) as errors in CI to enforce documentation.
- Include code examples in XML docs for key public APIs.

### Swagger / OpenAPI

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new() { Title = "My API", Version = "v1" });
    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
});
```

## Architecture Decision Records (ADRs)

For significant design decisions, create an ADR in `docs/adr/`:

```markdown
# ADR-001: Use MediatR for CQRS

## Status
Accepted

## Context
We need a consistent way to separate command and query responsibilities across API endpoints.

## Decision
Use MediatR for in-process command/query dispatching following the CQRS pattern.

## Consequences
- All request handlers implement `IRequestHandler<TRequest, TResponse>`.
- Pipeline behaviors handle cross-cutting concerns (validation, logging, caching).
- Dependency on MediatR NuGet package is required.
```

## Diagrams

- Use **Mermaid** diagrams in Markdown for architecture and flow documentation.
- Keep diagrams close to the code they describe.
- For complex systems, include both high-level and component-level diagrams.

## Notes for LLM Agent Behavior

- Always generate XML documentation comments for new public types, methods, and properties.
- Include `<summary>`, `<param>`, `<returns>`, and `<exception>` tags in method documentation.
- Add `<example>` blocks when the method behavior is non-obvious.
- When modifying existing code, update the corresponding XML documentation.
- Use `<inheritdoc/>` for interface implementations when the base documentation is sufficient.
- Do not generate redundant documentation that restates obvious type information.
- When generating API controllers, include `[ProducesResponseType]` attributes for Swagger documentation.
