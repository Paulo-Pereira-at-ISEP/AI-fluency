---
description: "Testing patterns and best practices for C# / .NET projects"
applyTo: "**/*.cs,**/Test*.cs,**/*Tests.cs,**/*Test.cs"
---

# Testing Best Practices

## High-Level Principles

- Every new feature or bug fix must include corresponding tests.
- Tests are first-class code: readable, maintainable, and well-organized.
- Aim for high coverage (>80%) but prioritize meaningful tests over coverage metrics.
- Tests should be fast, isolated, deterministic, and independent of execution order.

## Testing Framework

- Use **xUnit** as the primary testing framework (recommended by Microsoft; used in .NET runtime itself).
- Use **Coverlet** for code coverage reporting (`coverlet.collector` NuGet).
- Use **FluentAssertions** for expressive assertions.
- Use **NSubstitute** or **Moq** for mocking.
- Use **Bogus** for generating realistic test data.
- Use **Testcontainers** for integration tests requiring databases or services.
- Use **Verify** or **Snapshooter** for snapshot testing.
- Use **Architect.xUnit** for architecture tests.

## Test Organization

### Directory Structure

```
solution/
├── src/
│   ├── MyApp.Core/
│   │   ├── Services/
│   │   │   └── UserService.cs
│   │   └── Models/
│   │       └── User.cs
│   ├── MyApp.Api/
│   │   └── Controllers/
│   │       └── UsersController.cs
│   └── MyApp.Infrastructure/
│       └── Repositories/
│           └── SqlUserRepository.cs
├── tests/
│   ├── MyApp.Core.Tests/
│   │   └── Services/
│   │       └── UserServiceTests.cs
│   ├── MyApp.Api.Tests/
│   │   └── Controllers/
│   │       └── UsersControllerTests.cs
│   ├── MyApp.Integration.Tests/
│   │   ├── Fixtures/
│   │   │   └── DatabaseFixture.cs
│   │   └── Repositories/
│   │       └── SqlUserRepositoryTests.cs
│   └── MyApp.Architecture.Tests/
│       └── ArchitectureTests.cs
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Test projects | `<ProjectName>.Tests` | `MyApp.Core.Tests` |
| Test classes | `<ClassUnderTest>Tests` | `UserServiceTests` |
| Test methods | `<Method>_<Scenario>_<Expected>` | `GetById_WithValidId_ReturnsUser` |
| Fixtures | `<Description>Fixture` | `DatabaseFixture`, `WebAppFixture` |
| Builders / Factories | `<Type>Builder` | `UserBuilder`, `OrderBuilder` |

### Test Naming Pattern

Use descriptive names that express **what is being tested**, **under what condition**, and **what is expected**:

```csharp
// Pattern: Method_Scenario_ExpectedBehavior
public class UserServiceTests
{
    [Fact]
    public async Task GetByIdAsync_WithValidId_ReturnsUser() { ... }

    [Fact]
    public async Task GetByIdAsync_WithNonExistentId_ReturnsNull() { ... }

    [Fact]
    public async Task CreateAsync_WithDuplicateEmail_ThrowsConflictException() { ... }

    [Fact]
    public async Task DeleteAsync_WhenUnauthorized_ThrowsPermissionException() { ... }
}
```

## Writing Tests

### Basic Test Structure (Arrange-Act-Assert)

```csharp
public class DiscountCalculatorTests
{
    [Fact]
    public void Calculate_WithValidPriceAndRate_ReturnsDiscountedPrice()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        var price = 100.0m;
        var rate = 0.2m;

        // Act
        var result = calculator.Calculate(price, rate);

        // Assert
        result.Should().Be(80.0m);
    }
}
```

### Fixtures (Shared Setup)

```csharp
// Class-level fixture (shared across tests in one class)
public class UserServiceTests : IAsyncLifetime
{
    private readonly IUserRepository _repository;
    private readonly UserService _sut; // System Under Test

    public UserServiceTests()
    {
        _repository = Substitute.For<IUserRepository>();
        _sut = new UserService(_repository, NullLogger<UserService>.Instance);
    }

    public Task InitializeAsync() => Task.CompletedTask;
    public Task DisposeAsync() => Task.CompletedTask;
}

// Collection fixture (shared across multiple test classes)
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

public class DatabaseFixture : IAsyncLifetime
{
    public string ConnectionString { get; private set; } = string.Empty;

    public async Task InitializeAsync()
    {
        // Start test database container
        var container = new MsSqlBuilder().Build();
        await container.StartAsync();
        ConnectionString = container.GetConnectionString();
    }

    public async Task DisposeAsync()
    {
        // Cleanup
    }
}
```

### Theory / Parametrized Tests

```csharp
[Theory]
[InlineData(100.0, 0.0, 100.0)]
[InlineData(100.0, 0.1, 90.0)]
[InlineData(100.0, 0.5, 50.0)]
[InlineData(100.0, 1.0, 0.0)]
[InlineData(0.0, 0.5, 0.0)]
public void Calculate_WithVariousInputs_ReturnsExpectedResult(
    double price, double rate, double expected)
{
    var result = _calculator.Calculate((decimal)price, (decimal)rate);
    result.Should().Be((decimal)expected);
}

// MemberData for complex test cases
public static IEnumerable<object[]> InvalidInputCases =>
[
    [(-1.0), 0.1, "price"],
    [(100.0), -0.1, "rate"],
    [(100.0), 1.5, "rate"],
];

[Theory]
[MemberData(nameof(InvalidInputCases))]
public void Calculate_WithInvalidInput_ThrowsArgumentOutOfRange(
    double price, double rate, string paramName)
{
    var act = () => _calculator.Calculate((decimal)price, (decimal)rate);
    act.Should().Throw<ArgumentOutOfRangeException>()
       .WithParameterName(paramName);
}
```

### Mocking

```csharp
public class NotificationServiceTests
{
    private readonly IEmailSender _emailSender = Substitute.For<IEmailSender>();
    private readonly NotificationService _sut;

    public NotificationServiceTests()
    {
        _sut = new NotificationService(_emailSender);
    }

    [Fact]
    public async Task Send_WithValidUser_DelegatesToEmailSender()
    {
        // Arrange
        var user = new User { Email = "alice@example.com", Name = "Alice" };
        _emailSender.SendAsync(Arg.Any<EmailMessage>(), Arg.Any<CancellationToken>())
            .Returns(true);

        // Act
        var result = await _sut.SendNotificationAsync(user, "Hello");

        // Assert
        result.Should().BeTrue();
        await _emailSender.Received(1).SendAsync(
            Arg.Is<EmailMessage>(m =>
                m.To == user.Email &&
                m.Body == "Hello"),
            Arg.Any<CancellationToken>());
    }
}
```

### Testing Exceptions

```csharp
[Fact]
public async Task CreateAsync_WithInvalidEmail_ThrowsValidationException()
{
    // Arrange
    var request = new CreateUserRequest { Name = "Bob", Email = "not-an-email" };

    // Act
    var act = () => _sut.CreateAsync(request);

    // Assert
    await act.Should().ThrowAsync<ValidationException>()
        .WithMessage("*email*");
}
```

### Testing Async Code

```csharp
[Fact]
public async Task GetAllAsync_WhenCancelled_ThrowsOperationCancelled()
{
    // Arrange
    using var cts = new CancellationTokenSource();
    cts.Cancel();

    // Act
    var act = () => _sut.GetAllAsync(cts.Token);

    // Assert
    await act.Should().ThrowAsync<OperationCanceledException>();
}
```

### Builder Pattern for Test Data

```csharp
public class UserBuilder
{
    private string _name = "Alice";
    private string _email = "alice@example.com";
    private Role _role = Role.Viewer;

    public UserBuilder WithName(string name) { _name = name; return this; }
    public UserBuilder WithEmail(string email) { _email = email; return this; }
    public UserBuilder WithRole(Role role) { _role = role; return this; }

    public User Build() => new()
    {
        Name = _name,
        Email = new EmailAddress(_email),
        Role = _role,
    };

    public static implicit operator User(UserBuilder b) => b.Build();
}

// Usage in tests
var admin = new UserBuilder().WithName("Admin").WithRole(Role.Admin).Build();
```

## Test Categories

### Unit Tests

- Test individual classes and methods in isolation.
- Must be fast (<100ms per test) and have no external dependencies.
- Mock all I/O, network, and database calls.
- Target: cover all branches and edge cases of business logic.

### Integration Tests

- Test interactions between real components (API + database, service + message broker).
- Use **Testcontainers** for databases and external services.
- Use `WebApplicationFactory<T>` for ASP.NET Core integration tests.
- Mark with `[Trait("Category", "Integration")]` for selective execution.

```csharp
public class UsersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UsersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    [Trait("Category", "Integration")]
    public async Task GetUsers_ReturnsOkWithList()
    {
        var response = await _client.GetAsync("/api/v1/users");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

### Architecture Tests

```csharp
public class ArchitectureTests
{
    [Fact]
    public void Core_ShouldNotDependOn_Infrastructure()
    {
        var result = Types.InAssembly(typeof(UserService).Assembly)
            .ShouldNot()
            .HaveDependencyOn("MyApp.Infrastructure")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }
}
```

### Security Tests

Security tests verify that the application correctly defends against common attack vectors. Mark them with `[Trait("Category", "Security")]`.

```csharp
[Trait("Category", "Security")]
public class SecurityTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public SecurityTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Theory]
    [InlineData("<script>alert('xss')</script>")]
    [InlineData("'; DROP TABLE Users; --")]
    [InlineData("../../../etc/passwd")]
    [InlineData("\x00null_byte")]
    public async Task CreateUser_WithMaliciousInput_ReturnsBadRequest(string maliciousInput)
    {
        var request = new { Name = maliciousInput, Email = "test@example.com" };
        var response = await _client.PostAsJsonAsync("/api/v1/users", request);
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task ProtectedEndpoint_WithoutAuth_ReturnsUnauthorized()
    {
        var response = await _client.GetAsync("/api/v1/admin/settings");
        response.StatusCode.Should().BeOneOf(
            HttpStatusCode.Unauthorized,
            HttpStatusCode.Forbidden);
    }

    [Fact]
    public async Task InternalError_DoesNotLeakDetails()
    {
        var response = await _client.GetAsync("/api/v1/trigger-error");
        var body = await response.Content.ReadAsStringAsync();
        body.Should().NotContainAny("StackTrace", "Exception", "password", "connectionString");
    }

    [Fact]
    public async Task Response_ContainsSecurityHeaders()
    {
        var response = await _client.GetAsync("/");
        response.Headers.Should().ContainKey("X-Content-Type-Options");
        response.Headers.Should().ContainKey("X-Frame-Options");
    }
}
```

#### Security Test Categories

| Category | Focus | Examples |
|----------|-------|----------|
| Input validation | Reject malicious inputs | XSS, SQL injection, path traversal, overflows |
| Authentication | Auth enforcement | Missing tokens, expired tokens, invalid tokens |
| Authorization | Permission enforcement | Role escalation, accessing other users' data |
| Rate limiting | Abuse prevention | Brute-force login, API flooding |
| Error handling | No information leakage | Stack traces, file paths, connection strings |
| Headers | Security headers present | CSP, HSTS, X-Content-Type-Options |

## Test Configuration

### Test Project `.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="coverlet.collector" Version="6.*" />
    <PackageReference Include="FluentAssertions" Version="7.*" />
    <PackageReference Include="NSubstitute" Version="5.*" />
    <PackageReference Include="Bogus" Version="35.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\MyApp.Core\MyApp.Core.csproj" />
  </ItemGroup>
</Project>
```

### Running Tests

```bash
# Run all tests
dotnet test

# Run with coverage
dotnet test --collect:"XPlat Code Coverage"

# Run specific category
dotnet test --filter "Category=Integration"

# Run excluding slow/security tests
dotnet test --filter "Category!=Integration&Category!=Security"
```

## Best Practices Summary

1. **One concept per test**: Each test should verify exactly one behavior.
2. **Descriptive names**: `Method_Scenario_Expected` reads like a specification.
3. **No test interdependence**: Tests must not rely on execution order or shared mutable state.
4. **Fast feedback**: Unit tests should run in under a second; use traits to separate slow tests.
5. **Deterministic**: No flaky tests. Avoid reliance on timing, random data (unless seeded), or external services.
6. **DRY but readable**: Use fixtures and theories to reduce duplication, but keep individual tests easy to read.
7. **Test edge cases**: Empty inputs, null values, boundary values, very large inputs, error conditions.
8. **Security tests**: Include tests for input validation, authentication, authorization, rate limiting, and error information leakage.
9. **Maintain tests**: Delete obsolete tests; update tests when behavior changes intentionally.

## Notes for LLM Agent Behavior

- When generating new code, always suggest corresponding test classes and methods.
- Follow the Arrange-Act-Assert pattern in generated tests.
- Use FluentAssertions (`Should()`) for assertions in new test code.
- Use `[Theory]` with `[InlineData]` for testing the same logic with multiple inputs.
- Prefer NSubstitute for mocking in new projects.
- Include `CancellationToken` forwarding in async test methods.
- When generating API endpoint code, always suggest corresponding security tests (auth, input validation, error leakage).
- Use parametrized tests with malicious inputs (XSS, SQL injection, path traversal) to verify input validation.
- Always add type annotations and nullable reference types in test code.
