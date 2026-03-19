---
description: "Project structure, packaging, and dependency management conventions for C# / .NET"
applyTo: "**/*.cs,**/*.csproj,**/*.sln,**/*.props,**/*.targets"
---

# Project Structure & Dependency Management

## High-Level Principles

- Use a consistent, well-known solution layout across all repositories.
- `Directory.Build.props` is the central place for shared MSBuild properties.
- `global.json` pins the SDK version for reproducible builds.
- Use `.editorconfig` for consistent formatting across all projects.
- Keep NuGet package references centrally managed for version consistency.

## Standard Solution Layout

```
my-solution/
├── .github/
│   ├── copilot-instructions.md
│   ├── dependabot.yml
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── src/
│   ├── MyApp.Core/                    # Domain logic, no external dependencies
│   │   ├── Models/
│   │   ├── Services/
│   │   ├── Interfaces/
│   │   └── MyApp.Core.csproj
│   ├── MyApp.Application/             # Use cases, orchestration, CQRS handlers
│   │   ├── Commands/
│   │   ├── Queries/
│   │   ├── Validators/
│   │   └── MyApp.Application.csproj
│   ├── MyApp.Infrastructure/          # Data access, external services, I/O
│   │   ├── Repositories/
│   │   ├── ExternalServices/
│   │   └── MyApp.Infrastructure.csproj
│   └── MyApp.Api/                     # ASP.NET Core host, controllers, middleware
│       ├── Controllers/
│       ├── Middleware/
│       ├── Program.cs
│       ├── appsettings.json
│       └── MyApp.Api.csproj
├── tests/
│   ├── MyApp.Core.Tests/
│   ├── MyApp.Application.Tests/
│   ├── MyApp.Integration.Tests/
│   └── MyApp.Architecture.Tests/
├── docs/
│   ├── adr/
│   │   └── 001-use-mediatr.md
│   └── api/
├── scripts/
│   └── setup-local.ps1
├── .editorconfig
├── .gitignore
├── CHANGELOG.md
├── Directory.Build.props
├── Directory.Packages.props
├── global.json
├── LICENSE
├── Makefile
├── README.md
└── MyApp.sln
```

### Key Directory Roles

| Directory | Purpose |
|-----------|---------|
| `src/` | All production source code, one project per bounded context or layer |
| `tests/` | All test projects, mirroring src structure |
| `docs/` | Project documentation, ADRs, API docs |
| `scripts/` | Development and automation scripts |
| `.github/` | CI/CD workflows, Dependabot config, AI agent instructions |

### Architecture Layers

| Layer | Project | Depends On | Purpose |
|-------|---------|-----------|---------|
| Core / Domain | `MyApp.Core` | Nothing | Entities, value objects, domain services, interfaces |
| Application | `MyApp.Application` | Core | Use cases, CQRS handlers, DTOs, validators |
| Infrastructure | `MyApp.Infrastructure` | Core, Application | Database, APIs, file system, messaging |
| Presentation | `MyApp.Api` | Application, Infrastructure | HTTP host, controllers, middleware, DI setup |

## Central Configuration Files

### `global.json`

```json
{
  "sdk": {
    "version": "8.0.300",
    "rollForward": "latestFeature"
  }
}
```

### `Directory.Build.props`

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.*">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

### `Directory.Packages.props` (Central Package Management)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <!-- Core -->
    <PackageVersion Include="MediatR" Version="12.*" />
    <PackageVersion Include="FluentValidation" Version="11.*" />

    <!-- Infrastructure -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="8.*" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.*" />
    <PackageVersion Include="Dapper" Version="2.*" />

    <!-- API -->
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="6.*" />

    <!-- Testing -->
    <PackageVersion Include="xunit" Version="2.*" />
    <PackageVersion Include="FluentAssertions" Version="7.*" />
    <PackageVersion Include="NSubstitute" Version="5.*" />
    <PackageVersion Include="Bogus" Version="35.*" />
    <PackageVersion Include="Testcontainers" Version="3.*" />
    <PackageVersion Include="coverlet.collector" Version="6.*" />

    <!-- Security / Dev -->
    <PackageVersion Include="SecurityCodeScan.VS2019" Version="5.*" />
  </ItemGroup>
</Project>
```

### `.editorconfig` (Essential Rules)

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = crlf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.cs]
# Naming rules
dotnet_naming_rule.interface_should_begin_with_i.severity = error
dotnet_naming_rule.interface_should_begin_with_i.symbols = interface
dotnet_naming_rule.interface_should_begin_with_i.style = begins_with_i

dotnet_naming_rule.private_field_should_begin_with_underscore.severity = warning
dotnet_naming_rule.private_field_should_begin_with_underscore.symbols = private_field
dotnet_naming_rule.private_field_should_begin_with_underscore.style = underscore_prefix

# Code style
csharp_style_namespace_declarations = file_scoped:warning
csharp_style_var_for_built_in_types = false:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_prefer_primary_constructors = true:suggestion
csharp_using_directive_placement = outside_namespace:error

# Formatting
csharp_new_line_before_open_brace = all
csharp_new_line_before_else = true
csharp_new_line_before_catch = true
csharp_new_line_before_finally = true
```

## Dependency Management

### NuGet Best Practices

- Use **Central Package Management** (`Directory.Packages.props`) to unify versions across all projects.
- In individual `.csproj` files, reference packages without versions: `<PackageReference Include="MediatR" />`.
- Use `dotnet list package --outdated` to check for updates.
- Use `dotnet list package --vulnerable` to check for security vulnerabilities.
- Avoid `packages.config` — use `PackageReference` format exclusively.

### Restore & Build

```bash
# Restore all packages
dotnet restore

# Build the solution
dotnet build --configuration Release

# Publish for deployment
dotnet publish src/MyApp.Api -c Release -o ./publish
```

## CI/CD Conventions

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Format check
        run: dotnet format --verify-no-changes --no-restore

      - name: Test with coverage
        run: dotnet test --no-build --configuration Release --collect:"XPlat Code Coverage"

      - name: Security audit
        run: dotnet list package --vulnerable --include-transitive 2>&1 | tee audit.txt && ! grep -q "has the following vulnerable packages" audit.txt
```

### Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: nuget
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "security"
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
```

## Configuration Management

- Use `appsettings.json` for default configuration.
- Use `appsettings.{Environment}.json` for environment overrides.
- Use **User Secrets** (`dotnet user-secrets`) for local development secrets.
- Use **environment variables** for deployment-specific configuration.
- Use **Azure Key Vault** / **AWS Secrets Manager** for production secrets.
- Never commit secrets or credentials to the repository.
- Use `IOptions<T>` / `IOptionsSnapshot<T>` pattern for typed configuration.

```csharp
// Settings class
public class SmtpSettings
{
    public const string SectionName = "Smtp";
    public required string Host { get; init; }
    public required int Port { get; init; }
    public required string Username { get; init; }
    // Secret — loaded from User Secrets / Key Vault, never from appsettings.json
    public required string Password { get; init; }
}

// Registration
builder.Services.Configure<SmtpSettings>(builder.Configuration.GetSection(SmtpSettings.SectionName));
```

## Docker Support

```dockerfile
# Multi-stage build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.sln Directory.Build.props Directory.Packages.props global.json ./
COPY src/**/*.csproj ./src/
RUN dotnet restore
COPY . .
RUN dotnet publish src/MyApp.Api -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
# Run as non-root user
RUN adduser --disabled-password --gecos "" appuser
USER appuser
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

## Makefile / Task Runner

```makefile
.PHONY: restore build test lint format publish

restore:
	dotnet restore

build: restore
	dotnet build --no-restore -c Release

test: build
	dotnet test --no-build -c Release --collect:"XPlat Code Coverage"

lint:
	dotnet format --verify-no-changes

format:
	dotnet format

security-audit:
	dotnet list package --vulnerable --include-transitive

publish: build
	dotnet publish src/MyApp.Api -c Release -o ./publish --no-restore
```

## Notes for LLM Agent Behavior

- When creating new projects, follow the layered/Clean Architecture structure.
- Always generate a `Directory.Build.props` with nullable reference types, warnings as errors, and analyzers.
- Use Central Package Management (`Directory.Packages.props`) for multi-project solutions.
- Include `.editorconfig` with the standard naming rules.
- Suggest `Makefile` or equivalent for common development commands.
- Use `dotnet user-secrets` for local secrets — never `appsettings.json` for sensitive values.
- Always include security scanning steps (vulnerability audit, Dependabot) in CI workflows.
- Never generate code that commits secrets or credentials; use environment variables and User Secrets.
- Include the non-root user pattern in Dockerfile suggestions.
