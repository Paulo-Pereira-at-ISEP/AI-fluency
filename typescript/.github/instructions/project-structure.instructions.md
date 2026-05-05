---
description: "Project structure, configuration, dependency management, and CI/CD for TypeScript Node.js projects (REST APIs, libraries, CLI tools)."
---

# Project Structure — TypeScript / Node.js

## Directory Layout (REST API / Node.js)

```
my-api/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                    # Main CI pipeline
│   │   └── security-audit.yml        # Scheduled security scans
│   ├── dependabot.yml                # Automated dependency updates
│   ├── copilot-instructions.md
│   ├── typescript-coding.instructions.md
│   ├── documentation.instructions.md
│   ├── testing.instructions.md
│   ├── project-structure.instructions.md
│   ├── security.instructions.md
│   ├── code-review.agent.md
│   └── code-modernization.agent.md
├── docs/
│   ├── adr/                          # Architecture Decision Records
│   │   ├── 0001-use-zod-for-validation.md
│   │   └── template.md
│   └── openapi.yaml                  # OpenAPI 3.1 spec
├── src/
│   ├── app.ts                        # Express/Fastify app factory
│   ├── main.ts                       # Entry point (start server)
│   ├── config/
│   │   ├── env.ts                    # Zod-validated environment variables
│   │   └── database.ts               # DB connection setup
│   ├── shared/
│   │   ├── errors/
│   │   │   ├── app-error.ts          # Base error class
│   │   │   ├── http-errors.ts        # NotFoundError, ValidationError, etc.
│   │   │   └── index.ts
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts
│   │   │   ├── error-handler.middleware.ts
│   │   │   └── request-logger.middleware.ts
│   │   ├── types/
│   │   │   ├── express.d.ts          # Augmented Express types (req.user)
│   │   │   └── common.ts             # Shared type aliases (Result<T>, etc.)
│   │   └── utils/
│   │       ├── crypto.ts             # Secure random, hashing utilities
│   │       └── pagination.ts
│   └── modules/                      # Feature modules (one directory per domain)
│       ├── auth/
│       │   ├── auth.controller.ts
│       │   ├── auth.service.ts
│       │   ├── auth.routes.ts
│       │   ├── auth.schema.ts        # Zod schemas for this module
│       │   ├── auth.service.test.ts
│       │   └── index.ts              # Barrel export (public API only)
│       └── users/
│           ├── user.controller.ts
│           ├── user.service.ts
│           ├── user.repository.ts
│           ├── user.routes.ts
│           ├── user.model.ts         # Types and interfaces
│           ├── user.schema.ts        # Zod schemas
│           ├── user.service.test.ts
│           ├── user.repository.test.ts
│           └── index.ts
├── tests/
│   ├── integration/
│   │   ├── users.integration.test.ts
│   │   └── auth.integration.test.ts
│   └── helpers/
│       ├── test-app.ts
│       ├── test-db.ts
│       └── fixtures/
│           └── users.fixture.ts
├── .editorconfig
├── .eslintrc.json
├── .gitignore
├── .env.example                      # Committed — documents required env vars (no values)
├── .prettierrc
├── Dockerfile
├── docker-compose.yml                # Local dev with DB, Redis
├── Makefile
├── package.json
├── package-lock.json
├── tsconfig.json
├── tsconfig.build.json               # Excludes tests from production build
├── vitest.config.ts
├── vitest.integration.config.ts
└── README.md
```

## Directory Layout (TypeScript Library)

```
my-lib/
├── src/
│   ├── index.ts                      # Public API — only export what consumers need
│   ├── types.ts                      # Shared type definitions
│   └── utils/
│       ├── format.ts
│       └── format.test.ts
├── dist/                             # Build output (gitignored)
│   ├── index.js                      # ESM
│   ├── index.cjs                     # CJS (dual format)
│   └── index.d.ts                    # Type declarations
├── docs/
├── tsconfig.json
├── tsup.config.ts                    # Build configuration
├── vitest.config.ts
├── package.json
└── README.md
```

---

## TypeScript Configuration

### tsconfig.json (root — development)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "exactOptionalPropertyTypes": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "paths": {
      "@shared/*": ["./src/shared/*"],
      "@modules/*": ["./src/modules/*"],
      "@config/*": ["./src/config/*"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### tsconfig.build.json (production build — excludes tests)

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts", "tests/**"]
}
```

---

## ESLint Configuration (Flat Config)

```javascript
// eslint.config.mjs
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import security from 'eslint-plugin-security';
import imports from 'eslint-plugin-import';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    files: ['**/*.ts'],
    plugins: { security, import: imports },
    languageOptions: {
      parserOptions: {
        project: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      // TypeScript strict rules
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/explicit-function-return-type': ['error', { allowExpressions: true }],
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-misused-promises': 'error',
      '@typescript-eslint/await-thenable': 'error',
      '@typescript-eslint/no-non-null-assertion': 'warn',
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/consistent-type-exports': 'error',

      // Security
      'security/detect-object-injection': 'warn',
      'security/detect-non-literal-regexp': 'warn',
      'security/detect-unsafe-regex': 'error',
      'security/detect-eval-with-expression': 'error',
      'security/detect-non-literal-fs-filename': 'warn',
      'security/detect-child-process': 'warn',

      // Import ordering
      'import/order': ['error', {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        'newlines-between': 'always',
      }],
    },
  },
  {
    files: ['**/*.test.ts', '**/*.spec.ts', 'tests/**/*.ts'],
    rules: {
      '@typescript-eslint/no-explicit-any': 'off', // allow in test utilities
      '@typescript-eslint/explicit-function-return-type': 'off',
    },
  },
);
```

---

## Prettier Configuration

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "endOfLine": "lf"
}
```

---

## .editorconfig

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true
max_line_length = 100

[*.md]
trim_trailing_whitespace = false
```

---

## package.json

```json
{
  "name": "my-api",
  "version": "1.0.0",
  "type": "module",
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  },
  "scripts": {
    "build": "tsc --project tsconfig.build.json",
    "start": "node dist/main.js",
    "dev": "tsx watch src/main.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "lint": "eslint . --max-warnings=0",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write \"src/**/*.ts\" \"tests/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\" \"tests/**/*.ts\"",
    "typecheck": "tsc --noEmit",
    "audit": "npm audit --audit-level=high",
    "precommit": "lint-staged",
    "prepare": "husky"
  },
  "lint-staged": {
    "*.ts": ["eslint --fix", "prettier --write"]
  }
}
```

---

## Library Build — tsup

```typescript
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm', 'cjs'],        // dual format for maximum compatibility
  dts: true,                      // generate .d.ts declaration files
  sourcemap: true,
  clean: true,
  minify: false,                  // let consumers minify
  splitting: false,
  treeshake: true,
  target: 'node20',
  outDir: 'dist',
});
```

---

## GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  quality:
    name: Type Check, Lint & Test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npm run format:check

      - name: Unit tests with coverage
        run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

      - name: Integration tests
        run: npm run test:integration

      - name: Security audit
        run: npm audit --audit-level=high

      - name: Build
        run: npm run build
```

```yaml
# .github/workflows/security-audit.yml
name: Security Audit

on:
  schedule:
    - cron: '0 8 * * 1'   # every Monday at 08:00 UTC
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm audit --audit-level=moderate
```

---

## Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
    open-pull-requests-limit: 10
    groups:
      dev-dependencies:
        dependency-type: development
      production-dependencies:
        dependency-type: production
```

---

## Dockerfile (production)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts
COPY tsconfig*.json ./
COPY src ./src
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev --ignore-scripts
COPY --from=builder /app/dist ./dist
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

---

## Git Hooks (Husky + lint-staged)

```bash
# .husky/pre-commit
npm run typecheck
npm run lint
npx lint-staged
```

```bash
# .husky/commit-msg
npx --no -- commitlint --edit $1
```

```json
// commitlint.config.json
{
  "extends": ["@commitlint/config-conventional"]
}
```
