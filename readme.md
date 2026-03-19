# Instructions — LLM Agent Instruction Sets

## What is this repository?

This repository contains comprehensive instruction sets designed to be consumed by **LLM agents** (such as GitHub Copilot, Cursor, Cline, Aider, and others) during development sessions. Each subdirectory corresponds to a language/platform and provides rules, conventions, best practices, and security directives that the agent must follow when generating, reviewing, or modernising code.

The goal is to turn the agent into a team member that knows and respects project conventions from the very first prompt.

---

## Structure and Content

```
AI-fluency/
├── readme.md                        ← this file
├── python/
│   └── github/
│       ├── copilot-instructions.md          # Root instructions (architecture, commits, references)
│       ├── python-coding.instructions.md    # Python coding conventions (PEP 8, typing, async)
│       ├── documentation.instructions.md    # Docstrings, README, CHANGELOG, ADRs
│       ├── testing.instructions.md          # pytest, fixtures, security tests
│       ├── project-structure.instructions.md# src/ layout, pyproject.toml, CI/CD
│       ├── security.instructions.md         # OWASP Top 10, validation, auth, secrets
│       ├── code-review.agent.md             # Automated code review agent
│       └── code-modernization.agent.md      # Python 3.7 → 3.11+ migration agent
│
├── c-sharp/
│   └── github/
│       ├── copilot-instructions.md          # Root instructions (.NET 8, architecture, commits)
│       ├── csharp-coding.instructions.md    # C# 10–12 conventions (naming, DI, async, LINQ)
│       ├── documentation.instructions.md    # XML doc comments, Swagger, DocFX, ADRs
│       ├── testing.instructions.md          # xUnit, FluentAssertions, security tests
│       ├── project-structure.instructions.md# Clean Architecture, Central Package Mgmt, Docker
│       ├── security.instructions.md         # OWASP Top 10, ASP.NET Core hardening
│       ├── code-review.agent.md             # Automated code review agent
│       └── code-modernization.agent.md      # .NET Framework → .NET 8 migration agent
│
├── javascript/
│   └── github/
│       ├── copilot-instructions.md          # Root instructions (TypeScript, Angular 17+, commits)
│       ├── javascript-coding.instructions.md# TS/JS/Angular conventions (signals, DI, RxJS)
│       ├── documentation.instructions.md    # TSDoc, Compodoc, README, CHANGELOG, ADRs
│       ├── testing.instructions.md          # Jest, Angular TestBed, Playwright, security
│       ├── project-structure.instructions.md# Angular layout, ESLint flat config, CI/CD, Docker
│       ├── security.instructions.md         # OWASP Top 10, XSS, CSP, Angular security
│       ├── code-review.agent.md             # Automated code review agent
│       └── code-modernization.agent.md      # AngularJS/JS → Angular 17+/TS migration agent
│
└── java/
    └── github/
        ├── copilot-instructions.md          # Root instructions (Java 21, Spring Boot 3.x, commits)
        ├── java-coding.instructions.md      # Java 21 conventions (records, sealed, pattern matching)
        ├── documentation.instructions.md    # Javadoc, OpenAPI/Swagger, README, CHANGELOG, ADRs
        ├── testing.instructions.md          # JUnit 5, Mockito, AssertJ, Testcontainers, ArchUnit
        ├── project-structure.instructions.md# Hexagonal architecture, Maven/Gradle, CI/CD, Docker
        ├── security.instructions.md         # OWASP Top 10, Spring Security, deserialization, Log4Shell
        ├── code-review.agent.md             # Automated code review agent
        └── code-modernization.agent.md      # Java 8/11 → 21 + Spring Boot 2 → 3 migration agent
```

### File types

| Suffix | Purpose | When it is loaded |
|--------|---------|-------------------|
| `.instructions.md` | Rules and conventions the agent must follow | Automatically, as background context |
| `.agent.md` | Defines a specialised agent with inputs, outputs, and behaviour | When the user invokes the agent or requests a specific task |

---

## Why use instruction files?

### 1. Consistency
Without instructions, every agent response is an individual interpretation. With instructions, all generated code follows the same conventions — naming, formatting, error handling, security — as if a living style guide were supervising every line.

### 2. Security by default
The `security.instructions.md` files ensure the agent never generates code with known vulnerabilities (SQL injection, XSS, hardcoded secrets, `eval()`, `BinaryFormatter`, etc.), regardless of who writes the prompt.

### 3. Instant onboarding
New team members (human or agent) immediately absorb project conventions with no need for training or external documentation.

### 4. Reproducible quality
Code reviews and modernisations always follow the same criteria, eliminating variability between sessions.

### 5. Accumulated knowledge
The instructions act as a living repository of technical decisions (ADRs, library choices, architecture patterns) that evolves with the project.

---

## How to integrate into a project

### Step 1 — Copy the appropriate subdirectory

Copy the `github/` folder from the desired language subdirectory into the root of your repository under `.github/`:

```
# For a Python project
cp -r python/github/ <my-project>/.github/

# For a C# project
cp -r c-sharp/github/ <my-project>/.github/

# For a JavaScript / TypeScript / Angular project
cp -r javascript/github/ <my-project>/.github/

# For a Java / Spring Boot project
cp -r java/github/ <my-project>/.github/
```

The resulting structure in the target project will be:

```
my-project/
├── .github/
│   ├── copilot-instructions.md
│   ├── python-coding.instructions.md   (or csharp-coding / javascript-coding)
│   ├── documentation.instructions.md
│   ├── testing.instructions.md
│   ├── project-structure.instructions.md
│   ├── security.instructions.md
│   ├── code-review.agent.md
│   └── code-modernization.agent.md
├── src/
├── tests/
└── ...
```

### Step 2 — Customise

Edit the files to reflect the specifics of your project:

- **`copilot-instructions.md`** — Update the project name, concrete stack, references to internal modules.
- **`*-coding.instructions.md`** — Adjust naming rules, preferred libraries, minimum versions.
- **`security.instructions.md`** — Add organisation-specific policies (e.g. secrets provider, WAF, compliance requirements).
- **`project-structure.instructions.md`** — Update the directory tree and CI/CD pipelines.

### Step 3 — Commit

Commit the instruction files to the repository. They are code — they must be versioned, reviewed in PRs, and evolve with the project.

```bash
git add .github/*.md
git commit -m "docs: add LLM agent instructions for [Python|C#|JavaScript|Java]"
```

---

## How instruction files are passed as context to the LLM agent

### GitHub Copilot (VS Code / Visual Studio)

GitHub Copilot automatically loads instruction files placed in `.github/`:

| File | Behaviour |
|------|-----------|
| `.github/copilot-instructions.md` | Loaded **always** as background context in every interaction |
| `.github/*.instructions.md` | Loaded automatically when relevant to the task |
| `.github/*.agent.md` | Available as invocable agents (e.g. `@code-review`) |

> **Note**: The YAML frontmatter at the top of each file (the `description` field) helps Copilot decide when to include the file in the context.

### Cursor

In Cursor, place the files in the `.cursor/rules/` folder or reference them in `.cursorrules`:

```
.cursor/
└── rules/
    ├── copilot-instructions.md
    ├── python-coding.instructions.md
    └── ...
```

### Other agents (Cline, Aider, Continue, etc.)

Most agents accept instructions via:

1. **System files** — Configurable in the agent's settings (system prompt files).
2. **Direct reference** — Include the file in the prompt: `@file:.github/security.instructions.md`.
3. **Project context** — Some agents automatically scan `.md` files in the root or in `.github/`.

### Manual inclusion in the prompt

If the agent does not support automatic loading, you can always paste or reference the content:

```
Please follow the conventions defined in this file:

<instructions>
[content of the .instructions.md file]
</instructions>

Now, implement the following feature: ...
```

---

## Tips for effective use

### Keep files up to date
Outdated instructions are worse than none — they generate code that looks correct but does not follow current practices. Review the files every sprint or whenever there are significant changes to the project.

### Don't duplicate, reference
Each `.instructions.md` file covers a specific domain. If a file needs rules from another, reference it (`see security.instructions.md`) instead of copying the content.

### Start with `copilot-instructions.md`
This is the entry point. Make sure it contains a solid project overview and clear references to the specialised files.

### Use agents for concrete tasks
The `.agent.md` files define structured workflows. Use them for recurring tasks:
- **`@code-review`** before opening a PR.
- **`@code-modernization`** when it is time to update dependencies or migrate patterns.

### Version and review like code
Instruction files must go through the same review process as code. A poorly defined rule propagates to all agent-generated code.

### Adapt to the project context
The provided files are comprehensive templates. Remove what does not apply and add what is specific to your domain. Shorter, focused instructions are more effective than long, generic documents.

---

## Tips for more efficient prompts

### 1. Be specific about what you want

```
❌ "Create a user service"
✅ "Create a UserService that implements IUserService with async CRUD methods,
    using the repository pattern, with CancellationToken in all methods,
    following the conventions in csharp-coding.instructions.md"
```

### 2. Reference the relevant instruction files

```
✅ "Following the rules in security.instructions.md, review this controller
    and identify vulnerabilities"
```

The agent prioritises rules that are explicitly referenced in the prompt.

### 3. Break complex tasks down

Instead of requesting a complete feature in a single prompt, split it up:

```
Prompt 1: "Create the domain model for Order with properties X, Y, Z"
Prompt 2: "Create the IOrderRepository interface and the EF Core implementation"
Prompt 3: "Create the OrderService with validation using FluentValidation"
Prompt 4: "Create the unit tests for OrderService"
```

### 4. Ask for a review before accepting

```
✅ "Review the code you just generated using the rules in code-review.agent.md
    and fix any issues found"
```

### 5. Use the modernisation agent proactively

```
✅ "Analyse this file and suggest modernisations following
    code-modernization.agent.md"
```

### 6. Provide business context

```
✅ "This endpoint is public and internet-facing. Implement it following
    security.instructions.md with special attention to rate limiting and input validation"
```

The agent adjusts its rigour level based on the risk context.

### 7. Ask for explanations when needed

```
✅ "Explain why you chose this approach instead of X,
    referencing the relevant instructions"
```

This helps validate that the agent is effectively following the instructions.

### 8. Iterate with feedback

```
Prompt 1: "Implement X"
Prompt 2: "Method Y should use async/await with CancellationToken.
           Fix it following the conventions in csharp-coding.instructions.md"
```

Incremental corrections referencing specific instructions are more effective than rephrasing the entire request.

---

## Contributing

To add support for a new language or framework:

1. Create a new subdirectory at the repo root (e.g. `go/`, `rust/`, `kotlin/`).
2. Inside it, create the `github/` folder with files following the same structure.
3. Adapt all content to the target language, maintaining security coverage.
4. Update this `readme.md` with the new entry in the directory tree.

---

## Licence

These instruction files are internal to the project and intended for use by the development team and the LLM agents configured in the repository.
