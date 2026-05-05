---
description: "Project structure, configuration, dependency management, and CI/CD for JavaScript / TypeScript / Angular projects."
---

# Project Structure — JavaScript / TypeScript / Angular

## Directory Layout (Angular)

```
my-app/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                    # Main CI pipeline
│   │   └── security-audit.yml        # Scheduled security scans
│   ├── dependabot.yml                # Automated dependency updates
│   ├── copilot-instructions.md       # Root LLM instructions
│   ├── javascript-coding.instructions.md
│   ├── documentation.instructions.md
│   ├── testing.instructions.md
│   ├── project-structure.instructions.md
│   ├── security.instructions.md
│   ├── code-review.agent.md
│   └── code-modernization.agent.md
├── docs/
│   └── adr/                          # Architecture Decision Records
│       ├── 0001-use-standalone-components.md
│       └── template.md
├── src/
│   ├── app/
│   │   ├── core/                     # Singleton services, guards, interceptors
│   │   │   ├── guards/
│   │   │   │   └── auth.guard.ts
│   │   │   ├── interceptors/
│   │   │   │   ├── auth.interceptor.ts
│   │   │   │   └── error.interceptor.ts
│   │   │   ├── services/
│   │   │   │   ├── auth.service.ts
│   │   │   │   └── logger.service.ts
│   │   │   └── core.provider.ts      # Core dependency providers
│   │   ├── features/                 # Feature areas (lazy-loaded)
│   │   │   ├── dashboard/
│   │   │   │   ├── components/
│   │   │   │   ├── services/
│   │   │   │   ├── models/
│   │   │   │   └── dashboard.routes.ts
│   │   │   └── users/
│   │   │       ├── components/
│   │   │       │   ├── user-list/
│   │   │       │   │   ├── user-list.component.ts
│   │   │       │   │   ├── user-list.component.html
│   │   │       │   │   ├── user-list.component.scss
│   │   │       │   │   └── user-list.component.spec.ts
│   │   │       │   └── user-detail/
│   │   │       ├── services/
│   │   │       ├── models/
│   │   │       └── users.routes.ts
│   │   ├── shared/                   # Shared components, directives, pipes
│   │   │   ├── components/
│   │   │   │   ├── spinner/
│   │   │   │   └── error-message/
│   │   │   ├── directives/
│   │   │   ├── pipes/
│   │   │   ├── models/              # Shared interfaces & types
│   │   │   ├── utils/               # Pure utility functions
│   │   │   └── validators/          # Shared form validators
│   │   ├── app.component.ts
│   │   ├── app.config.ts            # Application providers
│   │   └── app.routes.ts            # Root route definitions
│   ├── assets/
│   │   ├── i18n/                    # Translation files
│   │   └── images/
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.prod.ts
│   ├── styles/
│   │   ├── _variables.scss
│   │   ├── _mixins.scss
│   │   └── styles.scss
│   ├── index.html
│   └── main.ts                      # Bootstrap with standalone API
├── tests/
│   ├── integration/
│   └── e2e/
│       ├── pages/                   # Page objects
│       └── specs/
├── .editorconfig
├── .eslintrc.json                   # Or eslint.config.mjs (flat config)
├── .prettierrc
├── .prettierignore
├── .gitignore
├── angular.json
├── package.json
├── package-lock.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.spec.json
├── jest.config.ts
├── playwright.config.ts
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── README.md
```

## Directory Purposes

| Directory | Contents | Import Alias |
|-----------|----------|-------------|
| `core/` | Singleton services, guards, interceptors, error handlers | `@core/*` |
| `features/` | Feature modules (lazy-loaded via routes) | Feature-relative |
| `shared/` | Reusable components, directives, pipes, utils | `@shared/*` |
| `shared/models/` | Interfaces, types, enums shared across features | `@shared/models/*` |
| `shared/utils/` | Pure utility/helper functions | `@shared/utils/*` |
| `environments/` | Environment-specific configuration | `@env/*` |
| `tests/e2e/` | Playwright E2E tests and page objects | — |

## tsconfig.json — Path Aliases

```json
{
  "compilerOptions": {
    "baseUrl": "src",
    "paths": {
      "@core/*": ["app/core/*"],
      "@shared/*": ["app/shared/*"],
      "@features/*": ["app/features/*"],
      "@env/*": ["environments/*"]
    }
  }
}
```

## TypeScript Configuration

### tsconfig.json (root)

```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "dom"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "exactOptionalPropertyTypes": true,
    "declaration": false,
    "downlevelIteration": true,
    "experimentalDecorators": true,
    "importHelpers": true,
    "sourceMap": true,
    "baseUrl": "src",
    "paths": {
      "@core/*": ["app/core/*"],
      "@shared/*": ["app/shared/*"],
      "@features/*": ["app/features/*"],
      "@env/*": ["environments/*"]
    }
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

## ESLint Configuration (Flat Config)

```javascript
// eslint.config.mjs
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import angular from '@angular-eslint/eslint-plugin';
import angularTemplate from '@angular-eslint/eslint-plugin-template';
import security from 'eslint-plugin-security';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    files: ['**/*.ts'],
    plugins: { '@angular-eslint': angular, security },
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/explicit-function-return-type': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@angular-eslint/prefer-standalone': 'error',
      '@angular-eslint/prefer-on-push-component-change-detection': 'error',
      'security/detect-object-injection': 'warn',
      'security/detect-non-literal-regexp': 'warn',
      'security/detect-unsafe-regex': 'error',
      'security/detect-eval-with-expression': 'error',
    },
  },
  {
    files: ['**/*.html'],
    plugins: { '@angular-eslint/template': angularTemplate },
    rules: {
      '@angular-eslint/template/no-negated-async': 'error',
      '@angular-eslint/template/prefer-control-flow': 'error',
    },
  },
);
```

## Prettier Configuration

```json
// .prettierrc
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

[Makefile]
indent_style = tab
```

## Package Management

### package.json — Dependency Organization

```json
{
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  },
  "scripts": {
    "start": "ng serve",
    "build": "ng build --configuration production",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:e2e": "playwright test",
    "lint": "eslint . --max-warnings=0",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write \"src/**/*.{ts,html,scss}\"",
    "format:check": "prettier --check \"src/**/*.{ts,html,scss}\"",
    "audit": "npm audit --audit-level=moderate",
    "audit:fix": "npm audit fix",
    "precommit": "lint-staged",
    "prepare": "husky"
  },
  "lint-staged": {
    "*.ts": ["eslint --fix", "prettier --write"],
    "*.html": ["prettier --write"],
    "*.scss": ["prettier --write"]
  }
}
```

### Dependency Categories

| Category | Examples | Notes |
|----------|----------|-------|
| **Angular core** | `@angular/core`, `@angular/router`, `@angular/forms` | Pin to same major version |
| **UI library** | `@angular/material`, `@angular/cdk` | Pin to Angular-compatible version |
| **State/Reactive** | `@ngrx/store`, `rxjs` | Prefer signals for new code |
| **HTTP/API** | `@angular/common/http` | Always use `HttpClient` |
| **i18n** | `@ngx-translate/core` | Or Angular built-in i18n |
| **Testing** | `jest`, `@testing-library/angular`, `playwright` | Dev dependencies only |
| **Linting** | `eslint`, `@typescript-eslint/*`, `@angular-eslint/*` | Dev dependencies only |
| **Security** | `helmet` (Node.js), `eslint-plugin-security` | Required |

## Git Hooks (Husky + lint-staged)

```bash
# .husky/pre-commit
npx lint-staged

# .husky/commit-msg
npx commitlint --edit $1
```

```javascript
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'build', 'ci', 'chore', 'security'],
    ],
  },
};
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
  build-and-test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [20]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npm run format:check

      - name: Unit & integration tests
        run: npm run test:ci

      - name: Build (production)
        run: npm run build

      - name: Security audit
        run: npm audit --audit-level=moderate

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage/lcov.info

  e2e:
    runs-on: ubuntu-latest
    needs: build-and-test

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: E2E tests
        run: npm run test:e2e

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
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

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: npm audit
        run: npm audit --audit-level=moderate

      - name: Check for known vulnerabilities
        run: npx audit-ci --moderate
```

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "team-leads"
    labels:
      - "dependencies"
    groups:
      angular:
        patterns:
          - "@angular/*"
          - "@angular-eslint/*"
      testing:
        patterns:
          - "jest*"
          - "@types/jest"
          - "playwright*"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Docker

### Dockerfile (Multi-stage)

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Serve with nginx
FROM nginx:alpine AS production

# Security: run as non-root
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

COPY --from=build /app/dist/my-app/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

# Security headers in nginx.conf
RUN chown -R appuser:appgroup /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown appuser:appgroup /var/run/nginx.pid

USER appuser
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf (with security headers)

```nginx
server {
    listen 8080;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "0" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self';" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Angular SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
```

## Makefile

```makefile
.PHONY: install dev build test lint format audit clean

install:
	npm ci

dev:
	ng serve --open

build:
	ng build --configuration production

test:
	npm run test:coverage

test-e2e:
	npx playwright test

lint:
	npm run lint

format:
	npm run format

audit:
	npm audit --audit-level=moderate

clean:
	rm -rf dist node_modules .angular coverage playwright-report

ci: install lint format test build audit
```

## LLM Agent Directives

When generating or modifying project structure, the agent MUST:

- Place new components in the correct feature directory under `features/`.
- Place shared/reusable code in `shared/` with proper barrel exports.
- Place singleton services, guards, and interceptors in `core/`.
- Use path aliases (`@core/`, `@shared/`, `@features/`) — never relative paths crossing module boundaries.
- Ensure all new routes are lazy-loaded via the route configuration.
- Add `data-testid` attributes to component templates for E2E testability.
- Include security-related dev dependencies (`eslint-plugin-security`, `audit-ci`) in any new project setup.
- Configure `npm audit` as a required CI step.
