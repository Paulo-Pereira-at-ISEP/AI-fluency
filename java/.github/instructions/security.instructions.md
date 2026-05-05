---
description: "Comprehensive security guidelines for Java projects with Spring Boot. Covers OWASP Top 10, Spring Security, input validation, and secure coding patterns."
---

# Security Guidelines — Java / Spring Boot

## Core Principles

1. **Security is not optional** — every feature must consider security from design to deployment.
2. **Defence in depth** — multiple layers of protection; never rely on a single control.
3. **Least privilege** — grant minimum necessary access to users, services, and code.
4. **Fail securely** — errors must not expose sensitive data or bypass security controls.
5. **Secure by default** — Spring Security's defaults must never be weakened without explicit review.

---

## 1. Input Validation

### Jakarta Bean Validation

```java
// ✅ Validate at controller boundaries
@PostMapping
public UserDto create(@Valid @RequestBody CreateUserRequest request) { ... }

// ✅ Comprehensive request validation
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100)
    @Pattern(regexp = "^[\\p{L}\\p{M}\\s'-]+$", message = "Contains invalid characters")
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
```

### Custom Validators

```java
// ✅ Custom constraint for safe HTML content
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = NoScriptTagsValidator.class)
public @interface NoScriptTags {
    String message() default "Content contains potentially dangerous HTML";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class NoScriptTagsValidator implements ConstraintValidator<NoScriptTags, String> {
    private static final Pattern DANGEROUS = Pattern.compile(
        "<script|javascript:|on\\w+\\s*=", Pattern.CASE_INSENSITIVE);

    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true;
        return !DANGEROUS.matcher(value).find();
    }
}
```

### Rules

- **Always use `@Valid`** on `@RequestBody`, `@PathVariable` validation, and cascaded objects.
- **Whitelist, don't blacklist** — define allowed characters/patterns.
- **Validate type, length, range, and format** for every input.
- **Reject unexpected fields** — use records (immutable, no extra fields parsed).
- **Validate at every boundary** — controllers, message consumers, external API responses.

---

## 2. SQL Injection Prevention

### JPA / Hibernate (Safe by Default)

```java
// ✅ SAFE: JPA named queries
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// ✅ SAFE: Spring Data derived queries
Optional<User> findByEmailIgnoreCase(String email);

// ✅ SAFE: Criteria API
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);
query.where(cb.equal(root.get("email"), email));

// ❌ DANGEROUS: String concatenation in JPQL
@Query("SELECT u FROM User u WHERE u.email = '" + email + "'") // ❌ SQL INJECTION!

// ❌ DANGEROUS: Native query with concatenation
em.createNativeQuery("SELECT * FROM users WHERE id = " + id); // ❌ SQL INJECTION!

// ✅ SAFE: Native query with parameters
em.createNativeQuery("SELECT * FROM users WHERE id = :id", User.class)
    .setParameter("id", id);
```

### JDBC (When Required)

```java
// ❌ NEVER use Statement with string concatenation
Statement stmt = conn.createStatement();
stmt.executeQuery("SELECT * FROM users WHERE id = " + id); // ❌

// ✅ ALWAYS use PreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setLong(1, id);
ResultSet rs = ps.executeQuery();
```

### Rules

- **Use JPA/Hibernate** parameterized queries for all database operations.
- **Never concatenate** user input into JPQL, HQL, or native SQL.
- **Never use `Statement`** — always use `PreparedStatement` for raw JDBC.
- **Use Spring Data repositories** (derived queries) whenever possible.

---

## 3. Authentication & Authorization

### Spring Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler()))
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.DELETE, "/api/v1/users/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // ✅ High cost factor
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        var config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com")); // ✅ Explicit
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        config.setMaxAge(86400L);

        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### Method-Level Security

```java
// ✅ Role-based access control
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

// ✅ Permission-based with SpEL
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserDto getUser(@P("userId") Long userId) { ... }

// ✅ Post-authorization (filter results)
@PostFilter("filterObject.ownerId == authentication.principal.id")
public List<Order> getOrders() { ... }
```

### JWT Token Handling

```java
// ✅ Token generation with proper claims
public String generateToken(UserDetails user) {
    return Jwts.builder()
        .subject(user.getUsername())
        .claim("roles", user.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority).toList())
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + TOKEN_EXPIRATION_MS))
        .signWith(getSigningKey(), Jwts.SIG.HS256)
        .compact();
}

// ✅ Token validation with proper error handling
public Claims validateToken(String token) {
    try {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    } catch (ExpiredJwtException e) {
        throw new AuthenticationException("Token expired");
    } catch (JwtException e) {
        throw new AuthenticationException("Invalid token");
    }
}
```

### Rules

- **Never disable CSRF** without explicit justification (stateless JWT APIs are an exception).
- **Use `BCryptPasswordEncoder`** with cost factor ≥ 12 for password hashing.
- **Use short-lived JWTs** (15 min) with refresh tokens.
- **Apply `@PreAuthorize`** on every service method that requires authorization.
- **Never hardcode** roles in controllers — use method security or request matchers.
- **Validate tokens** on every request in the filter chain.

---

## 4. Secrets Management

```java
// 🚫 NEVER hardcode secrets
private static final String JWT_SECRET = "my-secret-key"; // ❌
private static final String DB_PASSWORD = "admin123";     // ❌

// ✅ Use environment variables / Spring externalized configuration
@ConfigurationProperties(prefix = "app.security")
public record SecurityProperties(
    @NotBlank String jwtSecret,
    @Min(60) int tokenExpirationSeconds
) {}

// ✅ application.yml with environment variable references
// app:
//   security:
//     jwt-secret: ${JWT_SECRET}
//     token-expiration-seconds: 900
```

### Rules

- **Never commit secrets** to version control.
- **Use Spring profiles** (`application-dev.yml`, `application-prod.yml`) with external configuration.
- **Use a secrets manager** in production (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, Spring Cloud Vault).
- **Rotate secrets** regularly and immediately after any suspected compromise.
- **Add `.env` and secrets files** to `.gitignore`.
- **Use `@ConfigurationProperties`** with validation — never `@Value` for secrets.

---

## 5. Deserialization Security

```java
// 🚫 NEVER use Java's native serialization with untrusted data
ObjectInputStream ois = new ObjectInputStream(untrustedInput); // ❌
Object obj = ois.readObject(); // Remote Code Execution risk!

// ✅ Use JSON (Jackson) with strict typing
// Jackson is safe by default — it does not resolve arbitrary types
ObjectMapper mapper = JsonMapper.builder()
    .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES) // Reject extra fields alternatively
    .build();

// 🚫 NEVER enable default typing with untrusted input
mapper.enableDefaultTyping(); // ❌ Allows arbitrary class instantiation

// ✅ If polymorphic types are needed, use explicit annotations
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public sealed interface Animal permits Dog, Cat {}
```

### Rules

- **Never use `ObjectInputStream`** on untrusted data.
- **Never enable Jackson default typing** (`enableDefaultTyping()`).
- **Use sealed interfaces** with `@JsonTypeInfo` for polymorphic deserialization.
- **Prefer records** for DTOs — they cannot be manipulated via reflection as easily.

---

## 6. HTTP Security Headers

### Spring Security Headers

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .contentTypeOptions(Customizer.withDefaults())      // X-Content-Type-Options: nosniff
            .frameOptions(frame -> frame.deny())                 // X-Frame-Options: DENY
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true))
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'"))
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .permissionsPolicy(perms -> perms
                .policy("camera=(), microphone=(), geolocation=()")))
        // ... rest of config
        .build();
}
```

### Required Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS |
| `Content-Security-Policy` | `default-src 'self'; ...` | Prevent XSS/injection |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Control referrer leakage |
| `Permissions-Policy` | `camera=(), microphone=()` | Restrict browser features |

---

## 7. CORS Configuration

```java
// ❌ DANGEROUS: Allow all origins
config.setAllowedOrigins(List.of("*")); // Never in production!

// ✅ Explicit allowed origins
var config = new CorsConfiguration();
config.setAllowedOrigins(List.of(
    "https://app.example.com",
    "https://admin.example.com"
));
config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
config.setAllowCredentials(true);
config.setMaxAge(86400L);
```

### Rules

- **Never use wildcard `*` origins** in production.
- **Explicitly list** allowed origins, methods, and headers.
- **Set `allowCredentials(true)`** only when cookies/auth are needed.

---

## 8. Error Handling & Information Leakage

```yaml
# application.yml — NEVER expose internals
server:
  error:
    include-stacktrace: never
    include-message: never
    include-binding-errors: never
    include-exception: false
```

```java
// ✅ Global exception handler — never expose internals
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ProblemDetail handleUnexpected(Exception ex) {
        String errorId = UUID.randomUUID().toString();
        log.error("Unhandled error errorId={}", errorId, ex);

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problem.setTitle("Internal Error");
        problem.setDetail("An internal error occurred. Reference: " + errorId);
        // ❌ Never: problem.setDetail(ex.getMessage());
        // ❌ Never: problem.setProperty("stackTrace", ex.getStackTrace());
        return problem;
    }
}
```

### Rules

- **Configure `server.error.include-stacktrace: never`** in every environment.
- **Return generic error messages** with a correlation ID for support.
- **Log full errors server-side** for debugging.
- **Use ProblemDetail** (RFC 7807) for all error responses.

---

## 9. Sensitive Data in Logs

```java
// ❌ Logging sensitive data
log.info("User login: email={}, password={}", email, password);
log.debug("Request headers: {}", request.getHeaders()); // May contain Authorization

// ✅ Redact sensitive fields
log.info("User login: email={}", email);
log.debug("API request: method={}, uri={}", request.getMethod(), request.getRequestURI());

// ✅ Use Logback masking pattern in logback-spring.xml
// Or use a custom Logback converter to redact sensitive patterns
```

### Sensitive Fields to Never Log

- Passwords, password hashes
- API keys, tokens, secrets, JWT values
- Credit card numbers, CVVs
- Social security numbers, national IDs
- Full HTTP Authorization headers
- Session cookies
- Database connection strings with credentials

---

## 10. Rate Limiting

### Spring Boot Rate Limiting

```java
// ✅ Using Bucket4j with Spring Boot
@Configuration
public class RateLimitConfig {

    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        var filter = new RateLimitFilter();
        var registration = new FilterRegistrationBean<>(filter);
        registration.addUrlPatterns("/api/*");
        return registration;
    }
}

// ✅ Or use Spring Cloud Gateway rate limiting
// spring.cloud.gateway.routes[0].filters[0]=RequestRateLimiter
```

### Rules

- **Apply rate limiting** to all public API endpoints.
- **Use stricter limits** for authentication, password reset, and registration.
- **Return `429 Too Many Requests`** with `Retry-After` header.
- **Use distributed rate limiting** (Redis) in multi-instance deployments.

---

## 11. Dependency Security

### OWASP Dependency-Check

```xml
<!-- Maven plugin -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.3</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
    </configuration>
</plugin>
```

```bash
# Run manually
./mvnw dependency-check:check

# In CI — fail on high/critical CVEs
./mvnw dependency-check:check -DfailBuildOnCVSS=7
```

### Rules

- **Run OWASP Dependency-Check** in every CI pipeline.
- **Configure Dependabot** for automated security PRs.
- **Review CVE advisories** weekly and patch within SLA.
- **Pin dependency versions** — never use dynamic ranges.
- **Audit new dependencies** before adding — check maintenance status, license, known issues.

---

## 12. Cryptography

```java
// ✅ Use BCrypt or Argon2 for password hashing
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode(rawPassword);
boolean matches = encoder.matches(rawPassword, hash);

// ✅ Use SecureRandom for security-sensitive random values
SecureRandom random = SecureRandom.getInstanceStrong();
byte[] tokenBytes = new byte[32];
random.nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);

// ❌ Never use java.util.Random for security
Random insecure = new Random();
int otp = insecure.nextInt(999999); // ❌ Predictable!

// ❌ Never implement custom cryptography
public static String myEncrypt(String data, String key) { ... } // ❌

// ✅ Use standard libraries (JCA/JCE)
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
```

### Rules

- **Use `BCryptPasswordEncoder`** or `Argon2PasswordEncoder` for passwords.
- **Use `SecureRandom`** for all security-sensitive random values.
- **Never use `java.util.Random`** or `Math.random()` for tokens, keys, or OTPs.
- **Never implement custom crypto** — use JCA/JCE or Bouncy Castle.
- **Use AES-GCM** for symmetric encryption — never ECB mode.

---

## 13. File Upload Security

```java
@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {
    // ✅ Validate file size
    if (file.getSize() > 5 * 1024 * 1024) {
        throw new FileTooLargeException("File exceeds 5MB limit");
    }

    // ✅ Validate content type
    String contentType = file.getContentType();
    if (!Set.of("image/jpeg", "image/png", "image/webp").contains(contentType)) {
        throw new InvalidFileTypeException("Only JPEG, PNG, and WebP images are allowed");
    }

    // ✅ Generate safe filename — never use original
    String safeFilename = UUID.randomUUID() + getExtension(file.getOriginalFilename());

    // ✅ Store outside web root
    Path destination = uploadDir.resolve(safeFilename);
    if (!destination.normalize().startsWith(uploadDir.normalize())) {
        throw new SecurityException("Path traversal attempt detected");
    }

    file.transferTo(destination);
    return ResponseEntity.ok(safeFilename);
}
```

### Rules

- **Validate content type AND file extension** — both can be spoofed, but together are harder to bypass.
- **Limit file size** with `spring.servlet.multipart.max-file-size` and explicit validation.
- **Never use the original filename** — generate a UUID-based name.
- **Store outside the web root** or in cloud storage (S3, Azure Blob).
- **Check for path traversal** — normalize and verify the destination path.

---

## 14. Logging & Monitoring (Log4Shell Prevention)

```java
// ✅ Use SLF4J parameterized logging — NEVER string concatenation
log.info("Processing order orderId={}", orderId);

// ❌ DANGEROUS: User input in log message (Log4Shell, log injection)
log.info("User logged in: " + username); // ❌ Log injection risk

// ✅ Sanitize anything from user input before logging
log.info("User logged in: username={}", sanitize(username));

private static String sanitize(String input) {
    return input.replaceAll("[\\n\\r\\t]", "_");
}
```

### Rules

- **Always use SLF4J parameterized logging** — never string concatenation.
- **Sanitize user input** before including in log messages.
- **Keep Log4j and Logback updated** — Log4Shell (CVE-2021-44228) was in Log4j 2.x.
- **Disable JNDI lookups** if using Log4j 2: `-Dlog4j2.formatMsgNoLookups=true`.

---

## 15. Actuator Security

```yaml
# ✅ Expose only safe endpoints
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
    env:
      enabled: false  # ❌ Never expose env in production
    beans:
      enabled: false
    configprops:
      enabled: false
```

### Rules

- **Never expose `/actuator/env`** — it leaks environment variables and secrets.
- **Restrict actuator endpoints** to `health`, `info`, `metrics`, `prometheus`.
- **Protect actuator** with authentication in production.
- **Use `show-details: when-authorized`** for health endpoint.

---

## OWASP Top 10 — Quick Reference

| # | Vulnerability | Prevention in Java/Spring |
|---|--------------|--------------------------|
| A01 | Broken Access Control | `@PreAuthorize`, role-based `SecurityFilterChain`, method security |
| A02 | Cryptographic Failures | BCrypt/Argon2, SecureRandom, AES-GCM, HTTPS everywhere |
| A03 | Injection (SQL, LDAP, Log) | JPA parameterized queries, Bean Validation, SLF4J parameterized logging |
| A04 | Insecure Design | Threat modeling, ADRs, hexagonal architecture, security reviews |
| A05 | Security Misconfiguration | Spring Security defaults, no actuator exposure, `stacktrace: never` |
| A06 | Vulnerable Components | OWASP Dependency-Check, Dependabot, Snyk |
| A07 | Auth Failures | Spring Security, BCrypt, short-lived JWTs, MFA |
| A08 | Data Integrity Failures | Signed JWTs, Flyway migrations, CI verification |
| A09 | Logging & Monitoring | SLF4J structured logging, ELK/Grafana, sanitized input |
| A10 | Server-Side Request Forgery | URL allowlisting, no user-controlled URLs in server HTTP client |

---

## LLM Agent Directives

When generating or modifying code, the agent MUST:

- Never generate SQL with string concatenation — always use JPA or `PreparedStatement`.
- Never use `ObjectInputStream.readObject()` on untrusted data.
- Never hardcode secrets, API keys, or credentials in source code.
- Never use `java.util.Random` for security-sensitive operations.
- Never expose actuator `/env`, `/beans`, or `/configprops` endpoints.
- Always add `@Valid` on all `@RequestBody` and validated parameters.
- Always use `BCryptPasswordEncoder` (cost ≥ 12) for password handling.
- Always configure `server.error.include-stacktrace: never`.
- Always use parameterized SLF4J logging — never concatenation.
- Always include `@PreAuthorize` on service methods requiring authorization.
- Flag any use of wildcard `*` in CORS origins as a **critical security issue**.
- Recommend OWASP Dependency-Check for every new Maven/Gradle project.
- Include security headers via Spring Security's `headers()` configuration.
