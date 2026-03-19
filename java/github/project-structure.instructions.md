---
description: "Project structure, build configuration, dependency management, and CI/CD for Java projects with Spring Boot and Maven/Gradle."
---

# Project Structure — Java

## Directory Layout (Maven, Hexagonal Architecture)

```
my-app/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                        # Main CI pipeline
│   │   └── security-audit.yml            # Scheduled security scans
│   ├── dependabot.yml                    # Automated dependency updates
│   ├── copilot-instructions.md           # Root LLM instructions
│   ├── java-coding.instructions.md
│   ├── documentation.instructions.md
│   ├── testing.instructions.md
│   ├── project-structure.instructions.md
│   ├── security.instructions.md
│   ├── code-review.agent.md
│   └── code-modernization.agent.md
├── docs/
│   └── adr/                              # Architecture Decision Records
│       ├── 0001-use-hexagonal-architecture.md
│       └── template.md
├── src/
│   ├── main/
│   │   ├── java/com/example/myapp/
│   │   │   ├── domain/                   # Domain layer (no framework deps)
│   │   │   │   ├── user/
│   │   │   │   │   ├── User.java         # Entity / Aggregate root
│   │   │   │   │   ├── Email.java        # Value object
│   │   │   │   │   ├── UserRole.java     # Enum
│   │   │   │   │   └── UserRepository.java  # Port (interface)
│   │   │   │   └── order/
│   │   │   ├── application/              # Application / Use-case layer
│   │   │   │   ├── user/
│   │   │   │   │   ├── UserService.java
│   │   │   │   │   ├── CreateUserRequest.java   # Input DTO (record)
│   │   │   │   │   └── UserDto.java             # Output DTO (record)
│   │   │   │   └── order/
│   │   │   ├── infrastructure/           # Adapters: DB, messaging, external APIs
│   │   │   │   ├── persistence/
│   │   │   │   │   ├── JpaUserRepository.java
│   │   │   │   │   └── UserJpaEntity.java
│   │   │   │   ├── messaging/
│   │   │   │   └── external/
│   │   │   ├── web/                      # Adapters: REST controllers, filters
│   │   │   │   ├── UserController.java
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   └── security/
│   │   │   │       ├── SecurityConfig.java
│   │   │   │       └── JwtAuthFilter.java
│   │   │   └── MyAppApplication.java     # Spring Boot main class
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/migration/             # Flyway migrations
│   │       │   ├── V1__create_users_table.sql
│   │       │   └── V2__add_roles_table.sql
│   │       └── logback-spring.xml
│   └── test/
│       └── java/com/example/myapp/
│           ├── domain/
│           │   └── user/
│           │       ├── UserTest.java
│           │       └── EmailTest.java
│           ├── application/
│           │   └── user/
│           │       └── UserServiceTest.java
│           ├── web/
│           │   └── UserControllerTest.java
│           ├── integration/
│           │   ├── UserRepositoryIntegrationTest.java
│           │   └── UserApiIntegrationTest.java
│           └── architecture/
│               └── ArchitectureTest.java  # ArchUnit tests
├── .editorconfig
├── .gitignore
├── checkstyle.xml
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── README.md
```

## Layer Responsibilities

| Layer | Package | Depends On | Contains |
|-------|---------|-----------|----------|
| **Domain** | `domain.*` | Nothing | Entities, value objects, repository interfaces (ports), domain events |
| **Application** | `application.*` | Domain | Use cases, services, DTOs (records), input/output ports |
| **Infrastructure** | `infrastructure.*` | Domain, Application | JPA implementations, messaging adapters, external API clients |
| **Web** | `web.*` | Application | REST controllers, filters, exception handlers, security config |

## Maven Configuration (pom.xml)

### Parent & Java Version

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<properties>
    <java.version>21</java.version>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <!-- Dependency versions (managed centrally) -->
    <testcontainers.version>1.20.0</testcontainers.version>
    <archunit.version>1.3.0</archunit.version>
    <spotbugs.version>4.8.6</spotbugs.version>
</properties>
```

### Key Dependencies

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>

    <!-- Documentation -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.6.0</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.tngtech.archunit</groupId>
        <artifactId>archunit-junit5</artifactId>
        <version>${archunit.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Build Plugins

```xml
<build>
    <plugins>
        <!-- Spring Boot -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <!-- Compiler settings -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <release>21</release>
                <compilerArgs>
                    <arg>-parameters</arg>
                </compilerArgs>
            </configuration>
        </plugin>

        <!-- Code formatting (Spotless) -->
        <plugin>
            <groupId>com.diffplug.spotless</groupId>
            <artifactId>spotless-maven-plugin</artifactId>
            <version>2.43.0</version>
            <configuration>
                <java>
                    <googleJavaFormat>
                        <version>1.22.0</version>
                        <style>AOSP</style>
                    </googleJavaFormat>
                    <importOrder/>
                    <removeUnusedImports/>
                </java>
            </configuration>
        </plugin>

        <!-- Static analysis (SpotBugs) -->
        <plugin>
            <groupId>com.github.spotbugs</groupId>
            <artifactId>spotbugs-maven-plugin</artifactId>
            <version>${spotbugs.version}</version>
            <configuration>
                <effort>Max</effort>
                <threshold>Medium</threshold>
            </configuration>
        </plugin>

        <!-- Code coverage (JaCoCo) -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>
                <execution>
                    <goals><goal>prepare-agent</goal></goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>verify</phase>
                    <goals><goal>report</goal></goals>
                </execution>
            </executions>
        </plugin>

        <!-- OWASP Dependency-Check -->
        <plugin>
            <groupId>org.owasp</groupId>
            <artifactId>dependency-check-maven</artifactId>
            <version>10.0.3</version>
            <configuration>
                <failBuildOnCVSS>7</failBuildOnCVSS>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## Gradle Alternative (Kotlin DSL)

```kotlin
// build.gradle.kts
plugins {
    java
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.6"
    id("com.diffplug.spotless") version "6.25.0"
    id("org.owasp.dependencycheck") version "10.0.3"
    jacoco
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

// Version catalog: gradle/libs.versions.toml
```

## Application Configuration

### application.yml

```yaml
spring:
  application:
    name: my-app
  threads:
    virtual:
      enabled: true  # Java 21 virtual threads
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    open-in-view: false  # Always disable OSIV
    hibernate:
      ddl-auto: validate  # Flyway handles migrations
    properties:
      hibernate:
        default_batch_fetch_size: 20
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  shutdown: graceful
  error:
    include-stacktrace: never  # Security: never expose stack traces
    include-message: never

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized

logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

## CI/CD — GitHub Actions

### Main CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Format check
        run: ./mvnw spotless:check

      - name: Build & unit tests
        run: ./mvnw clean verify -DskipITs

      - name: Integration tests
        run: ./mvnw verify -Pintegration-test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

      - name: SpotBugs analysis
        run: ./mvnw spotbugs:check

      - name: OWASP Dependency-Check
        run: ./mvnw dependency-check:check

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: target/site/jacoco/jacoco.xml
```

### Security Audit Pipeline

```yaml
# .github/workflows/security-audit.yml
name: Security Audit

on:
  schedule:
    - cron: '0 8 * * 1'  # Every Monday at 08:00 UTC
  workflow_dispatch:

permissions:
  contents: read
  security-events: write

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: OWASP Dependency-Check
        run: ./mvnw dependency-check:check -DfailBuildOnCVSS=4

      - name: SpotBugs security analysis
        run: ./mvnw spotbugs:check
```

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "team-leads"
    labels:
      - "dependencies"
    groups:
      spring:
        patterns:
          - "org.springframework*"
      testing:
        patterns:
          - "org.testcontainers*"
          - "com.tngtech.archunit*"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Docker

### Dockerfile (Multi-stage)

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B

COPY src/ src/
RUN ./mvnw clean package -DskipTests -B

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

# Security: run as non-root
USER appuser

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_DATASOURCE_USERNAME: myuser
      SPRING_DATASOURCE_PASSWORD: mypass
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

## Makefile

```makefile
.PHONY: build test test-int lint format audit clean run

build:
	./mvnw clean package -DskipTests

run:
	./mvnw spring-boot:run

test:
	./mvnw test

test-int:
	./mvnw verify -Pintegration-test

lint:
	./mvnw spotbugs:check checkstyle:check

format:
	./mvnw spotless:apply

format-check:
	./mvnw spotless:check

audit:
	./mvnw dependency-check:check

clean:
	./mvnw clean

ci: format-check test lint audit build
```

## .editorconfig

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true
max_line_length = 120

[*.{yml,yaml}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab

[pom.xml]
indent_size = 4
```

## LLM Agent Directives

When generating or modifying project structure, the agent MUST:

- Place domain logic in the `domain` package with zero framework imports.
- Place services and DTOs in the `application` package.
- Place JPA entities, repository implementations, and external adapters in `infrastructure`.
- Place controllers, filters, and security config in `web`.
- Use `@ConfigurationProperties` records for all configuration — never `@Value` with magic strings.
- Add Flyway migration scripts for every schema change — never use `ddl-auto: create/update`.
- Ensure `spring.jpa.open-in-view: false` in every application.yml.
- Ensure `server.error.include-stacktrace: never` in every application.yml.
- Include OWASP Dependency-Check and SpotBugs in every Maven/Gradle build.
- Configure Dependabot for automated dependency updates.
- Use multi-stage Docker builds with non-root user.
