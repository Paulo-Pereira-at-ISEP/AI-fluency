---
description: "Coding conventions and best practices for Java development with Spring Boot."
---

# Java — Coding Conventions

## General Principles

- Target **Java 21 LTS** — use modern features (records, sealed classes, pattern matching, virtual threads).
- Follow **Google Java Style Guide** as the baseline, enforced via `google-java-format` + Spotless.
- Prefer **immutability** — `final` fields, records, `List.of()`, `Map.of()`, `Collections.unmodifiable*()`.
- Keep methods **short and focused** — a method should do one thing.
- **No raw types** — always parameterize generics.
- **No checked exceptions** in domain logic — wrap in unchecked domain exceptions at boundaries.

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Packages | `lowercase.dotted` | `com.example.users.service` |
| Classes, interfaces, enums, records | `PascalCase` | `UserService`, `OrderStatus` |
| Methods, variables, parameters | `camelCase` | `getUserById()`, `userName` |
| Constants (`static final`) | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Type parameters | Single uppercase letter or short name | `T`, `E`, `K`, `V`, `ID` |
| Test classes | `PascalCase` + `Test` suffix | `UserServiceTest` |
| Test methods | `camelCase` descriptive | `shouldReturnUserWhenIdExists()` |
| Spring beans | `camelCase` (implicit from class name) | `userService`, `authFilter` |
| REST endpoints | `kebab-case` | `/api/v1/user-profiles` |
| Database tables/columns | `snake_case` | `user_profiles`, `created_at` |

## Modern Java Features (Java 17–21)

### Records (Java 16+)

```java
// ✅ Use records for immutable data carriers (DTOs, value objects)
public record UserDto(
    Long id,
    String name,
    String email,
    Instant createdAt
) {}

// ✅ Record with compact constructor for validation
public record Email(String value) {
    public Email {
        Objects.requireNonNull(value, "Email must not be null");
        if (!value.matches("^[\\w.-]+@[\\w.-]+\\.\\w{2,}$")) {
            throw new IllegalArgumentException("Invalid email format: " + value);
        }
        value = value.toLowerCase().strip();
    }
}

// ❌ Don't use records for mutable entities or JPA entities
```

### Sealed Classes (Java 17+)

```java
// ✅ Use sealed classes for restricted type hierarchies
public sealed interface PaymentResult
    permits PaymentSuccess, PaymentFailure, PaymentPending {
}

public record PaymentSuccess(String transactionId, Instant timestamp) implements PaymentResult {}
public record PaymentFailure(String errorCode, String message) implements PaymentResult {}
public record PaymentPending(String referenceId) implements PaymentResult {}

// ✅ Exhaustive switch with pattern matching
String describe(PaymentResult result) {
    return switch (result) {
        case PaymentSuccess s -> "Paid: " + s.transactionId();
        case PaymentFailure f -> "Failed: " + f.message();
        case PaymentPending p -> "Pending: " + p.referenceId();
    };
}
```

### Pattern Matching (Java 21)

```java
// ✅ Pattern matching for instanceof (Java 16+)
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
}

// ✅ Pattern matching for switch (Java 21)
double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> 0.5 * t.base() * t.height();
    };
}

// ✅ Guarded patterns
String classify(Integer value) {
    return switch (value) {
        case Integer i when i < 0 -> "negative";
        case Integer i when i == 0 -> "zero";
        case Integer i when i > 0 -> "positive";
        default -> "unknown"; // unreachable but required
    };
}
```

### Text Blocks (Java 15+)

```java
// ✅ Use text blocks for multi-line strings
String query = """
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.active = true
    ORDER BY u.name
    """;

String json = """
    {
        "name": "%s",
        "email": "%s"
    }
    """.formatted(name, email);
```

### Virtual Threads (Java 21)

```java
// ✅ Use virtual threads for I/O-bound tasks
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<UserDto>> futures = userIds.stream()
        .map(id -> executor.submit(() -> userService.getById(id)))
        .toList();

    List<UserDto> results = futures.stream()
        .map(f -> {
            try { return f.get(); }
            catch (Exception e) { throw new RuntimeException(e); }
        })
        .toList();
}

// ✅ Spring Boot 3.2+ with virtual threads
// application.yml: spring.threads.virtual.enabled: true
```

### Helpful NullPointerExceptions & Optional

```java
// ✅ Use Optional for return types that may be absent
public Optional<User> findByEmail(String email) {
    return userRepository.findByEmail(email);
}

// ✅ Chain Optional operations
String displayName = findByEmail(email)
    .map(User::getDisplayName)
    .orElse("Unknown User");

// ❌ Never use Optional as a field, method parameter, or collection element
// ❌ Never call .get() without .isPresent() — use orElse/orElseThrow
```

## Spring Boot Patterns

### Constructor Injection

```java
// ✅ Constructor injection (implicit @Autowired for single constructor)
@Service
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final UserMapper userMapper;

    // Single constructor — Spring auto-injects
    public UserService(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder,
            UserMapper userMapper) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.userMapper = userMapper;
    }
}

// ❌ Never use field injection
@Service
public class BadService {
    @Autowired private UserRepository userRepository; // ❌
}
```

### Configuration Properties

```java
// ✅ Type-safe configuration with validation
@ConfigurationProperties(prefix = "app.security")
@Validated
public record SecurityProperties(
    @NotBlank String jwtSecret,
    @Min(60) int tokenExpirationSeconds,
    @NotEmpty List<String> allowedOrigins
) {}

// Enable in main class or config
@EnableConfigurationProperties(SecurityProperties.class)

// ❌ Never read properties with magic strings
// environment.getProperty("app.security.jwt-secret") // ❌
```

### REST Controllers

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getById(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto create(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }

    @PutMapping("/{id}")
    public UserDto update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        return userService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### Exception Handling

```java
// ✅ Domain exceptions (unchecked)
public class NotFoundException extends RuntimeException {
    public NotFoundException(String entity, Object id) {
        super("%s with id '%s' not found".formatted(entity, id));
    }
}

public class BusinessRuleException extends RuntimeException {
    private final String code;

    public BusinessRuleException(String code, String message) {
        super(message);
        this.code = code;
    }

    public String getCode() { return code; }
}

// ✅ Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ProblemDetail handleNotFound(NotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Error");
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                e -> e.getDefaultMessage() != null ? e.getDefaultMessage() : "Invalid",
                (a, b) -> a
            ));
        problem.setProperty("errors", errors);
        return problem;
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        // ❌ Never expose stack traces or internal details
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An internal error occurred."
        );
    }
}
```

### Logging

```java
// ✅ Use SLF4J with parameterized messages
private static final Logger log = LoggerFactory.getLogger(UserService.class);

log.info("Creating user with email={}", email);
log.debug("User details: name={}, roles={}", name, roles);
log.error("Failed to process order orderId={}", orderId, exception);

// ❌ Never use string concatenation
log.info("Creating user: " + email); // ❌ Allocates even if level is disabled

// ❌ Never log sensitive data
log.info("Login attempt: email={}, password={}", email, password); // ❌
```

## Input Validation (Jakarta Bean Validation)

```java
// ✅ Use Jakarta Bean Validation annotations
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100)
    String name,

    @NotBlank @Email
    String email,

    @NotBlank @Size(min = 8, max = 128)
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).*$",
             message = "Must contain uppercase, lowercase, and digit")
    String password,

    @NotNull
    UserRole role
) {}

// ✅ Custom validator
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = SafeHtmlValidator.class)
public @interface SafeHtml {
    String message() default "Contains unsafe HTML content";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

## Collections & Streams

```java
// ✅ Use unmodifiable collections
List<String> names = List.of("Alice", "Bob", "Charlie");
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87);
Set<String> tags = Set.of("java", "spring");

// ✅ Use Stream API with clear, readable pipelines
List<UserDto> activeUsers = users.stream()
    .filter(User::isActive)
    .sorted(Comparator.comparing(User::getName))
    .map(userMapper::toDto)
    .toList(); // Java 16+ — prefer over .collect(Collectors.toList())

// ✅ Use Collectors for complex aggregations
Map<UserRole, List<User>> byRole = users.stream()
    .collect(Collectors.groupingBy(User::getRole));

// ❌ Don't use streams for simple iterations with side effects
// Use a for-each loop instead
for (User user : users) {
    notificationService.sendWelcome(user);
}
```

## Formatting & Style

- **google-java-format** via Spotless plugin handles all formatting.
- **4-space indentation** (Google style uses 2, but Spring ecosystem commonly uses 4 — align with project `checkstyle.xml`).
- **Max line length**: 120 characters.
- **Braces**: always use braces, even for single-line `if`/`for`/`while`.
- **Imports**: no wildcard imports (`import java.util.*` → ❌); organize alphabetically.

## Security

> See `security.instructions.md` for comprehensive guidelines.

Essential rules for every Java file:

1. **Never concatenate user input** into SQL queries — use parameterized queries or JPA.
2. **Never use `ObjectInputStream.readObject()`** on untrusted data without filtering.
3. **Never hardcode** passwords, API keys, or secrets in source code.
4. **Always validate** user input at controller boundaries with `@Valid`.
5. **Always use** Spring Security for authentication and authorization.
6. **Always use** parameterized logging — never concatenate user input in log messages.
7. **Never expose** stack traces or internal details in API error responses.
8. **Always use** `PreparedStatement` if writing raw JDBC — never `Statement` with string concat.
9. **Always run** OWASP Dependency-Check in CI.
10. **Never disable** CSRF protection without explicit justification.

## LLM Agent Directives

When generating or modifying Java code, the agent MUST:

- Target Java 21 — use records, sealed classes, pattern matching, text blocks where appropriate.
- Use constructor injection — never `@Autowired` on fields.
- Use `final` on all fields, parameters, and local variables where possible.
- Add `@Valid` on all `@RequestBody` and validated parameters.
- Return `ProblemDetail` (RFC 7807) for all error responses.
- Use SLF4J parameterized logging — never string concatenation.
- Follow the import ordering convention — no wildcard imports.
- Add `@Override` on all overriding methods.
- Use `Optional` for nullable return types — never return `null` from a method that could return Optional.
- Validate security implications of every generated code block against `security.instructions.md`.
