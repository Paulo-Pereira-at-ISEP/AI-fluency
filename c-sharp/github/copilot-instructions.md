---
description: "Comprehensive guidelines for AI coding agents in C# / .NET projects"
applyTo: "**/*.cs,**/*.csx,**/*.csproj,**/*.sln"
---

# C# Copilot Instructions

## Quick Reference

- **Style**: Follow the .NET coding conventions and Microsoft's C# style guide; use `dotnet format` for formatting.
- **Type Safety**: Use strong typing, nullable reference types (`#nullable enable`), and avoid `dynamic` / `object` where possible.
- **Naming**: `PascalCase` for public members and types, `camelCase` for local variables and parameters, `_camelCase` for private fields.
- **Error Handling**: Use specific exceptions; avoid catching `System.Exception` without re-throwing.
- **Security**: Validate all inputs, never hardcode secrets, use parameterized queries. See security instructions.
- **Testing**: Write tests for all new functionality using `xUnit` (or `NUnit`/`MSTest`).
- **Commit Messages**: Follow the Conventional Commits specification (see below).

## Project Architecture

- **Typical Structure**:
  - `src/` : Application source code, organized by project/domain.
  - `tests/` : Unit and integration tests mirroring the `src/` structure.
  - `docs/` : Documentation (DocFX, Markdown).
  - `scripts/` : Build and automation scripts.
  - Root config files: `Directory.Build.props`, `global.json`, `.editorconfig`.
- **Service Boundaries**:
  - Communication between components uses interfaces, dependency injection, and well-defined contracts.
  - External integrations are isolated behind repository/adapter/gateway patterns.
- **Data Flows**:
  - Data is managed through strongly-typed models, DTOs, and record types.
  - I/O operations are separated from business logic following Clean Architecture or Vertical Slice patterns.

## Developer Workflows

- **Building / Running**:
  - Use `dotnet build` and `dotnet run` from the CLI.
  - Use `Directory.Build.props` for shared MSBuild properties across projects.
  - Use `global.json` to pin the SDK version.
- **Testing**:
  - Run tests with `dotnet test`. Use `coverlet` for coverage reports.
  - Refer to the testing instructions for detailed patterns and conventions.
- **Linting & Formatting**:
  - Use `.editorconfig` for consistent code style across editors.
  - Use `dotnet format` for automated formatting.
  - Use Roslyn analyzers and `StyleCop.Analyzers` for code quality enforcement.

## Commit Message Template

```
<type>(<scope>): <short summary in imperative mood, max 72 chars>

<body: what changed and why, wrapped at 72 characters>

Tests:
- List tests performed or added.

Closes: #<issue-number>
```

### Conventional Commit Types

| Type | Usage |
|------|-------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation changes |
| `style` | Formatting, no logic change |
| `refactor` | Code restructuring, no behavior change |
| `test` | Adding or updating tests |
| `chore` | Build, CI, tooling changes |
| `perf` | Performance improvements |
| `security` | Security fixes or improvements |

## References

- **C# Coding Guidelines**: [csharp-coding.instructions.md](csharp-coding.instructions.md)
- **Documentation Standards**: [documentation.instructions.md](documentation.instructions.md)
- **Testing Best Practices**: [testing.instructions.md](testing.instructions.md)
- **Project Structure**: [project-structure.instructions.md](project-structure.instructions.md)
- **Application Security**: [security.instructions.md](security.instructions.md)

## Feedback

If any section is unclear or incomplete, please provide feedback to improve this document.
