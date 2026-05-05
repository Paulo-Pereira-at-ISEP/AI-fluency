---
description: "Application security guidelines and best practices for C# / .NET projects"
applyTo: "**/*.cs,**/*.csproj,**/*.json,**/*.yml,**/*.yaml"
---

# Application Security Guidelines

## High-Level Principles

- Security is not optional — it must be embedded in every phase of development, not added as an afterthought.
- Adopt a **defense-in-depth** strategy: multiple layers of protection so that a failure in one does not compromise the system.
- Apply the **principle of least privilege**: code, services, and users should have only the minimum permissions needed.
- Assume all external input is hostile until validated.
- Keep the attack surface minimal: disable unused features, remove dead code, limit exposed endpoints.

---

## 1. Input Validation & Sanitization

### Rules

- **Validate all external input** at the boundary (API endpoints, CLI arguments, file uploads, configuration).
- Use **allowlists** (whitelist) over denylists (blacklist) for validation.
- Use **Data Annotations**, **FluentValidation**, or **Pydantic-style validators** for structured input validation.
- Validate data types, ranges, lengths, formats, and allowed characters.
- Never trust client-side validation alone.

### Patterns

```csharp
using FluentValidation;

public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .MinimumLength(3)
            .MaximumLength(30)
            .Matches(@"^[a-zA-Z0-9_]+$")
            .WithMessage("Username must contain only letters, digits, and underscores.");

        RuleFor(x => x.Email)
            .NotEmpty()
            .MaximumLength(254)
            .EmailAddress();

        RuleFor(x => x.Age)
            .InclusiveBetween(13, 150);
    }
}
```

### Path Traversal Prevention

```csharp
public static string SafeReadFile(string baseDir, string userFilename)
{
    var basePath = Path.GetFullPath(baseDir);
    var fullPath = Path.GetFullPath(Path.Combine(basePath, userFilename));

    if (!fullPath.StartsWith(basePath, StringComparison.OrdinalIgnoreCase))
        throw new UnauthorizedAccessException("Access denied: path traversal detected.");

    return File.ReadAllText(fullPath);
}
```

### File Upload Validation

```csharp
private static readonly HashSet<string> AllowedExtensions = [".jpg", ".jpeg", ".png", ".pdf"];
private const long MaxFileSize = 10 * 1024 * 1024; // 10 MB

public static void ValidateUpload(IFormFile file)
{
    var ext = Path.GetExtension(file.FileName).ToLowerInvariant();
    if (!AllowedExtensions.Contains(ext))
        throw new ValidationException($"File extension '{ext}' is not allowed.");

    if (file.Length > MaxFileSize)
        throw new ValidationException($"File exceeds maximum size of {MaxFileSize} bytes.");

    // Never rely solely on ContentType from the client — verify server-side.
}
```

---

## 2. Authentication & Authorization

### Rules

- Never implement custom authentication — use ASP.NET Core Identity, OAuth 2.0, or OpenID Connect.
- Use **Argon2**, **bcrypt**, or **PBKDF2** for password hashing — never MD5, SHA-1, or plain SHA-256.
- Enforce strong password policies at the application level.
- Implement account lockout or rate limiting after failed login attempts.
- Use short-lived tokens (JWT) with proper expiration, refresh, and rotation.
- Always verify authorization on the server side for every request.

### ASP.NET Core Authentication

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromMinutes(1),
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
        };
    });

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"))
    .AddPolicy("EditorOrAbove", policy => policy.RequireRole("Editor", "Admin"));
```

### Authorization on Endpoints

```csharp
[ApiController]
[Route("api/v1/[controller]")]
[Authorize] // All endpoints require authentication by default
public class UsersController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "EditorOrAbove")]
    public async Task<IActionResult> GetAll(CancellationToken ct) { ... }

    [HttpDelete("{id}")]
    [Authorize(Policy = "AdminOnly")]
    public async Task<IActionResult> Delete(string id, CancellationToken ct) { ... }
}
```

### Password Hashing

```csharp
using Microsoft.AspNetCore.Identity;

// Use the built-in PasswordHasher (PBKDF2 with 600,000 iterations by default in .NET 8+)
var hasher = new PasswordHasher<User>();
string hash = hasher.HashPassword(user, plainPassword);
PasswordVerificationResult result = hasher.VerifyHashedPassword(user, hash, plainPassword);
```

---

## 3. Secrets Management

### Rules

- **Never** hardcode secrets, API keys, tokens, passwords, or connection strings in source code.
- **Never** commit secrets to version control — even in private repositories.
- Use **User Secrets** (`dotnet user-secrets`) for local development.
- Use **Azure Key Vault**, **AWS Secrets Manager**, or **HashiCorp Vault** for production.
- Rotate secrets regularly and on suspected compromise.
- Add `appsettings.*.json` with secrets to `.gitignore`.

### User Secrets (Development)

```bash
# Initialize user secrets for a project
dotnet user-secrets init --project src/MyApp.Api

# Set a secret
dotnet user-secrets set "Jwt:Key" "my-super-secret-key" --project src/MyApp.Api
```

### Azure Key Vault (Production)

```csharp
// Program.cs
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

### Pre-Commit Secret Detection

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

---

## 4. SQL Injection & Database Security

### Rules

- **Never** construct SQL queries with string concatenation or interpolation.
- **Always** use parameterized queries, ORMs with proper binding, or stored procedures.
- Use the principle of least privilege for database credentials.
- Validate and sanitize all user input before passing to queries.

### Patterns

```csharp
// DANGEROUS — SQL injection vulnerability
var query = $"SELECT * FROM Users WHERE Email = '{userEmail}'"; // NEVER DO THIS

// SAFE — Parameterized query (ADO.NET)
using var cmd = new SqlCommand("SELECT * FROM Users WHERE Email = @Email", connection);
cmd.Parameters.AddWithValue("@Email", userEmail);

// SAFE — Dapper
var user = await connection.QueryFirstOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Email = @Email",
    new { Email = userEmail });

// SAFE — Entity Framework Core
var user = await context.Users.FirstOrDefaultAsync(u => u.Email == userEmail, ct);
```

---

## 5. Cross-Site Scripting (XSS) Prevention

### Rules

- ASP.NET Core Razor and Blazor encode output by default — do not use `@Html.Raw()` with user data.
- Set the `Content-Type` header correctly for all responses.
- Use Content Security Policy (CSP) headers to restrict script sources.
- For APIs returning HTML content, use `HtmlEncoder.Default.Encode()`.

```csharp
using System.Text.Encodings.Web;

var safeOutput = HtmlEncoder.Default.Encode(userInput);
```

---

## 6. Cross-Site Request Forgery (CSRF) Protection

### Rules

- Use ASP.NET Core's built-in anti-forgery tokens (`[ValidateAntiForgeryToken]`) for MVC/Razor Pages forms.
- For APIs, prefer token-based authentication (Bearer tokens) over cookie-based sessions.
- Set `SameSite` attribute on cookies (`Lax` or `Strict`).

```csharp
// Auto anti-forgery for all POST actions in MVC
builder.Services.AddControllersWithViews(options =>
{
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
});
```

---

## 7. HTTP Security Headers

### Middleware

```csharp
// SecurityHeadersMiddleware.cs
public class SecurityHeadersMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var headers = context.Response.Headers;
        headers["X-Content-Type-Options"] = "nosniff";
        headers["X-Frame-Options"] = "DENY";
        headers["X-XSS-Protection"] = "0";  // Disabled; use CSP instead
        headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains";
        headers["Content-Security-Policy"] = "default-src 'self'";
        headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
        headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";

        await next(context);
    }
}

// Program.cs
app.UseMiddleware<SecurityHeadersMiddleware>();
```

---

## 8. Dependency Security

### Rules

- Audit NuGet packages for known vulnerabilities regularly.
- Use Central Package Management to unify versions.
- Remove unused packages.
- Prefer well-maintained packages with active security response.

### Tools & Automation

```bash
# Check for vulnerable packages
dotnet list package --vulnerable --include-transitive

# Update packages
dotnet outdated  # (requires dotnet-outdated-tool)
```

### Dependabot

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
```

### CI Security Scanning

```yaml
# Add to .github/workflows/ci.yml
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - name: Vulnerability audit
        run: dotnet list package --vulnerable --include-transitive 2>&1 | tee audit.txt && ! grep -q "has the following vulnerable packages" audit.txt
      - name: Security code analysis
        uses: security-code-scan/security-code-scan-results-action@v1
```

---

## 9. Cryptography

### Rules

- Use `System.Security.Cryptography.RandomNumberGenerator` for tokens, API keys, and nonces — **never** `System.Random`.
- Use AES-256-GCM for symmetric encryption.
- Use RSA-OAEP or ECDH for asymmetric encryption.
- Use the `Microsoft.AspNetCore.DataProtection` API for encrypting sensitive data at rest.
- Store encryption keys separately from encrypted data (Key Vault).
- Never roll your own cryptography.

### Patterns

```csharp
using System.Security.Cryptography;

// Generate cryptographically secure tokens
var apiKey = Convert.ToBase64String(RandomNumberGenerator.GetBytes(32));
var sessionId = Convert.ToHexString(RandomNumberGenerator.GetBytes(32));

// Data Protection API (recommended for ASP.NET Core)
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToAzureBlobStorage(/* ... */)
    .ProtectKeysWithAzureKeyVault(/* ... */);

// Usage
public class TokenService(IDataProtector protector)
{
    public string Protect(string plainText) => protector.Protect(plainText);
    public string Unprotect(string encrypted) => protector.Unprotect(encrypted);
}
```

---

## 10. Logging & Monitoring for Security

### Rules

- Log all authentication events (login, logout, failed attempts, password changes).
- Log all authorization failures (access denied events).
- Log all input validation failures from external sources.
- **Never** log sensitive data: passwords, tokens, API keys, PII, credit card numbers.
- Use structured logging for security events to enable automated alerting.
- Implement rate limiting and monitor for anomalous patterns.

### Patterns

```csharp
public static partial class SecurityLog
{
    [LoggerMessage(Level = LogLevel.Warning, Message = "Failed login attempt for user {UserId} from {IpAddress}: {Reason}")]
    public static partial void LoginFailed(ILogger logger, string? userId, string ipAddress, string reason);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Access denied for user {UserId} to resource {Resource}")]
    public static partial void AccessDenied(ILogger logger, string userId, string resource);

    [LoggerMessage(Level = LogLevel.Information, Message = "User {UserId} authenticated successfully from {IpAddress}")]
    public static partial void LoginSucceeded(ILogger logger, string userId, string ipAddress);
}
```

---

## 11. API Security

### Rules

- Always use HTTPS in production; enforce with HSTS.
- Implement **rate limiting** using `Microsoft.AspNetCore.RateLimiting`.
- Validate `Content-Type` headers on all incoming requests.
- Return generic error messages to clients; log detailed errors server-side.
- Use API versioning to manage breaking changes safely.
- Implement request size limits to prevent memory exhaustion.

### Rate Limiting

```csharp
// Program.cs (.NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", limiter =>
    {
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.PermitLimit = 100;
        limiter.QueueLimit = 0;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

// On controller/endpoint
[EnableRateLimiting("api")]
public class UsersController : ControllerBase { ... }
```

### CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins("https://yourdomain.com")  // NEVER use AllowAnyOrigin in production
              .WithMethods("GET", "POST")
              .WithHeaders("Authorization", "Content-Type")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromHours(1));
    });
});
```

### Request Size Limits

```csharp
// Limit request body size globally
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
});

// Or per-endpoint
[RequestSizeLimit(1_048_576)] // 1 MB
public async Task<IActionResult> Upload(IFormFile file) { ... }
```

---

## 12. Serialization & Deserialization

### Rules

- **Never** use `BinaryFormatter` — it is insecure and deprecated.
- Prefer `System.Text.Json` for JSON serialization; configure strict deserialization.
- Avoid deserializing untrusted data with `Newtonsoft.Json` `TypeNameHandling` enabled.
- Never use `Type.GetType()` or `Activator.CreateInstance()` with user input.

```csharp
// Safe JSON deserialization with strict options
var options = new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    // Do NOT set: JsonSerializerOptions with TypeInfoResolver that auto-resolves types
};

var result = JsonSerializer.Deserialize<MyModel>(jsonString, options);
```

---

## 13. Session & Cookie Security

### Rules

- Set `HttpOnly` flag on session cookies (prevents JavaScript access).
- Set `Secure` flag on cookies (transmitted only over HTTPS).
- Set `SameSite=Lax` or `SameSite=Strict` (prevents CSRF).
- Use short session expiration times and implement idle timeout.

```csharp
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Lax;
    options.ExpireTimeSpan = TimeSpan.FromHours(1);
    options.SlidingExpiration = true;
});
```

---

## 14. Error Handling for Security

### Rules

- Return generic error messages to clients; never expose stack traces, file paths, or internal details in production.
- Log the full exception server-side for debugging.
- Use exception handler middleware to transform internal errors into safe API responses.

```csharp
// Program.cs
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler(error =>
    {
        error.Run(async context =>
        {
            var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
            var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;

            logger.LogError(exception, "Unhandled exception at {Path}", context.Request.Path);

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(new
            {
                Detail = "An internal error occurred. Please try again later."
            });
        });
    });
}
```

---

## 15. Security Checklist for Code Review

Use this checklist when reviewing code for security:

- [ ] **Input validation**: All external inputs validated and sanitized
- [ ] **Authentication**: Auth checks present on all protected endpoints
- [ ] **Authorization**: Role/permission checks enforced server-side (`[Authorize]` attributes)
- [ ] **No hardcoded secrets**: No API keys, passwords, or tokens in code or `appsettings.json`
- [ ] **Parameterized queries**: No SQL string concatenation or interpolation
- [ ] **No dangerous APIs**: No `BinaryFormatter`, `Type.GetType()` with user data, `Process.Start()` with user input
- [ ] **Error handling**: No sensitive data leaked in error responses
- [ ] **Logging**: Security events logged; no sensitive data in logs
- [ ] **Dependencies**: No known vulnerabilities (`dotnet list package --vulnerable`)
- [ ] **Crypto**: Using `RandomNumberGenerator` for randomness; no weak algorithms
- [ ] **HTTPS**: All external communication over TLS; HSTS enabled
- [ ] **Headers**: Security headers set (CSP, HSTS, X-Content-Type-Options)
- [ ] **CORS**: Origins restricted to known domains
- [ ] **Rate limiting**: Abuse prevention on public endpoints
- [ ] **File handling**: Path traversal prevented; upload size/type limited
- [ ] **Nullable**: Nullable reference types enabled; null checks at boundaries

---

## OWASP Top 10 Quick Reference

| # | Risk | Key Mitigation |
|---|------|---------------|
| A01 | Broken Access Control | `[Authorize]` policies, least privilege, CORS |
| A02 | Cryptographic Failures | TLS everywhere, Data Protection API, no hardcoded keys |
| A03 | Injection | Parameterized queries, EF Core, FluentValidation |
| A04 | Insecure Design | Threat modeling, security requirements, secure defaults |
| A05 | Security Misconfiguration | Minimal permissions, security headers, no debug in prod |
| A06 | Vulnerable Components | `dotnet list package --vulnerable`, Dependabot |
| A07 | Authentication Failures | ASP.NET Core Identity, MFA, account lockout, rate limiting |
| A08 | Data Integrity Failures | Signed updates, CI/CD security, input validation |
| A09 | Logging Failures | Structured security logging, monitoring, alerting |
| A10 | SSRF | Validate URLs, allowlist destinations, `HttpClient` restrictions |

---

## Notes for LLM Agent Behavior

- **Always** generate input validation (FluentValidation or Data Annotations) for any endpoint that accepts external input.
- **Never** suggest `BinaryFormatter`, `Type.GetType()` with user input, or unparameterized SQL.
- **Always** use parameterized queries or EF Core — never string interpolation for SQL.
- **Always** use `RandomNumberGenerator` instead of `System.Random` for security-sensitive values.
- When generating API endpoints, include `[Authorize]`, input validation, rate limiting, and error handling that does not leak internals.
- When generating error handlers, ensure no sensitive information is leaked to clients.
- When suggesting NuGet packages, check for known vulnerabilities and prefer well-maintained packages.
- Flag any hardcoded credentials, keys, or tokens found in code and recommend User Secrets or Azure Key Vault.
- When generating logging code, ensure no passwords, tokens, or PII are included in log messages.
- Default to the most secure option when multiple approaches exist.
