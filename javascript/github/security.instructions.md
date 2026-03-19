---
description: "Comprehensive security guidelines for JavaScript, TypeScript, and Angular projects. Covers OWASP Top 10, Angular-specific security, and secure coding patterns."
---

# Security Guidelines — JavaScript / TypeScript / Angular

## Core Principles

1. **Security is not optional** — every feature must consider security from design to deployment.
2. **Defence in depth** — multiple layers of protection; never rely on a single control.
3. **Least privilege** — grant minimum necessary access to users, services, and code.
4. **Fail securely** — errors must not expose sensitive data or bypass security controls.
5. **Secure by default** — Angular's built-in protections must never be disabled without explicit review.

---

## 1. Input Validation & Sanitization

### Client-Side Validation (Angular)

```typescript
// ✅ Angular Reactive Forms with validators
this.userForm = this.fb.group({
  name: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(100)]],
  email: ['', [Validators.required, Validators.email]],
  age: [null, [Validators.required, Validators.min(0), Validators.max(150)]],
  website: ['', [Validators.pattern(/^https:\/\/[\w\-.]+(\.[\w\-.]+)+[/#?]?.*$/)]],
});

// ✅ Custom validator
function noScriptTags(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = /<script|javascript:|on\w+\s*=/i.test(control.value ?? '');
    return forbidden ? { dangerousContent: true } : null;
  };
}
```

### Server-Side Validation (Node.js/Express)

```typescript
// ✅ Zod for runtime validation
import { z } from 'zod';

const CreateUserSchema = z.object({
  name: z.string().min(2).max(100).trim(),
  email: z.string().email().toLowerCase(),
  age: z.number().int().min(0).max(150),
  role: z.enum(['admin', 'editor', 'viewer']),
});

type CreateUserDto = z.infer<typeof CreateUserSchema>;

// In route handler
app.post('/api/users', (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({
      error: 'Validation failed',
      details: result.error.flatten(),
    });
  }
  // Use result.data — validated & typed
});
```

### Rules

- **Always validate on the server** — client-side validation is for UX only.
- **Whitelist, don't blacklist** — define allowed characters/patterns, not disallowed ones.
- **Validate type, length, range, and format** for every input.
- **Trim and normalize** strings before processing.
- **Reject unexpected fields** — use strict schemas that disallow unknown properties.

---

## 2. Cross-Site Scripting (XSS) Prevention

### Angular Built-in Protections

Angular automatically sanitizes values bound to the DOM:

```typescript
// ✅ SAFE: Angular auto-escapes interpolation
<p>{{ user.name }}</p>                  // Text interpolation — auto-escaped
<img [src]="user.avatarUrl" />          // Property binding — sanitized

// ❌ DANGEROUS: Bypassing sanitization
<div [innerHTML]="userContent"></div>   // Angular sanitizes, but avoid when possible

// 🚫 NEVER do this without security review:
this.sanitizer.bypassSecurityTrustHtml(userInput);   // XSS risk!
this.sanitizer.bypassSecurityTrustScript(userInput);  // XSS risk!
this.sanitizer.bypassSecurityTrustUrl(userInput);     // XSS risk!
```

### Content Security Policy (CSP)

```html
<!-- In index.html or via HTTP header (preferred) -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data: https:;
               font-src 'self';
               connect-src 'self' https://api.example.com;
               frame-ancestors 'none';
               base-uri 'self';
               form-action 'self';">
```

### Rules

- **Never use `innerHTML`** with user-generated content — use text interpolation `{{ }}`.
- **Never call `bypassSecurityTrust*`** without a documented security review.
- **Never use `eval()`**, `Function()`, `setTimeout(string)`, or `setInterval(string)`.
- **Always configure CSP** via HTTP headers in production.
- **Sanitize** all content from external sources before rendering.

---

## 3. Cross-Site Request Forgery (CSRF)

### Angular CSRF Protection

```typescript
// ✅ Angular's built-in XSRF support
// In app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN',
      }),
    ),
  ],
};
```

### Rules

- **Use Angular's `withXsrfConfiguration()`** for all state-changing HTTP requests.
- **Set `SameSite=Strict` or `SameSite=Lax`** on session cookies.
- **Validate `Origin` and `Referer` headers** on the server for sensitive operations.

---

## 4. Authentication & Authorization

### Token Management

```typescript
// ✅ Store tokens in HTTP-only cookies (set by server)
// NEVER store tokens in localStorage or sessionStorage

// ❌ DANGEROUS: Token accessible to XSS
localStorage.setItem('token', jwt);

// ✅ SAFE: HTTP-only cookie (not accessible via JavaScript)
// Server sets: Set-Cookie: token=jwt; HttpOnly; Secure; SameSite=Strict; Path=/

// ✅ Token refresh pattern
@Injectable({ providedIn: 'root' })
export class AuthInterceptor implements HttpInterceptor {
  private readonly auth = inject(AuthService);

  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    return next.handle(req).pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse && error.status === 401) {
          return this.auth.refreshToken().pipe(
            switchMap(() => next.handle(req)),
          );
        }
        return throwError(() => error);
      }),
    );
  }
}
```

### Route Guards

```typescript
// ✅ Functional route guard
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url },
  });
};

// ✅ Role-based guard
export const roleGuard = (allowedRoles: string[]): CanActivateFn => {
  return () => {
    const auth = inject(AuthService);
    const currentRole = auth.currentUser()?.role;
    return currentRole != null && allowedRoles.includes(currentRole);
  };
};

// Usage in routes
export const routes: Routes = [
  {
    path: 'admin',
    canActivate: [authGuard, roleGuard(['admin'])],
    loadChildren: () => import('./features/admin/admin.routes'),
  },
];
```

### Rules

- **Never store tokens in `localStorage` or `sessionStorage`** — use HTTP-only cookies.
- **Always use route guards** for protected pages.
- **Implement token refresh** to avoid forcing re-login.
- **Validate tokens on the server** for every API request.
- **Use short-lived access tokens** (15 min) with longer-lived refresh tokens.

---

## 5. Secrets Management

```typescript
// 🚫 NEVER hardcode secrets
const API_KEY = 'sk-abc123def456';                    // ❌
const DB_URI = 'mongodb://admin:password@host:27017'; // ❌

// ✅ Use environment variables (server-side only)
const apiKey = process.env['API_KEY'];
if (!apiKey) {
  throw new Error('API_KEY environment variable is required');
}

// ✅ Angular environment files for non-secret config only
// environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com', // URL is not a secret
  // ❌ NEVER put API keys here — Angular bundles are public!
};
```

### Rules

- **Never commit secrets** to version control — use `.env` files (gitignored) for local dev.
- **Never include secrets in Angular bundles** — they are publicly downloadable.
- **API keys for third-party services** must be proxied through your backend.
- **Use a secrets manager** in production (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault).
- **Rotate secrets** regularly and immediately after any suspected compromise.
- **Use `detect-secrets`** as a pre-commit hook.

### .gitignore

```gitignore
# Secrets & environment
.env
.env.local
.env.*.local
*.pem
*.key
```

---

## 6. SQL Injection & NoSQL Injection

### Parameterized Queries

```typescript
// ❌ SQL Injection vulnerability
const query = `SELECT * FROM users WHERE id = '${req.params.id}'`;

// ✅ Parameterized query (pg)
const result = await pool.query('SELECT * FROM users WHERE id = $1', [req.params.id]);

// ✅ Prisma ORM (safe by default)
const user = await prisma.user.findUnique({ where: { id: userId } });

// ❌ NoSQL Injection (MongoDB)
const user = await db.collection('users').findOne({ name: req.body.name }); // Dangerous if name is { $gt: '' }

// ✅ Validate and type-check before querying
const name = CreateUserSchema.parse(req.body).name; // Validated string
const user = await db.collection('users').findOne({ name });
```

### Rules

- **Never interpolate user input** into SQL or NoSQL queries.
- **Use parameterized queries** or ORM methods exclusively.
- **Validate input types** before querying — ensure strings are strings, numbers are numbers.
- **Use ORMs** (Prisma, TypeORM, Drizzle) that parameterize by default.

---

## 7. Dependency Security

### npm Audit

```bash
# Check for known vulnerabilities
npm audit

# Fix automatically where possible
npm audit fix

# CI: fail on moderate+ vulnerabilities
npx audit-ci --moderate
```

### Dependabot / Renovate

- Configure Dependabot (see `project-structure.instructions.md`) for automated updates.
- Review security advisories weekly.
- Pin major versions; allow minor/patch auto-merge after CI passes.

### Lock File

- **Always commit `package-lock.json`** — ensures reproducible builds.
- **Use `npm ci`** in CI/CD — installs from lock file exactly.
- **Never run `npm install`** in CI — it may update the lock file.

### Supply Chain Attacks

```typescript
// ✅ Verify package provenance
// In .npmrc
audit=true
fund=false

// ✅ Use npm package provenance (if available)
npm install --prefer-offline
```

### Rules

- **Run `npm audit`** in every CI pipeline.
- **Configure Dependabot** with automatic PRs for security updates.
- **Review `npm audit` advisories** weekly and patch within SLA.
- **Pin exact versions** for critical dependencies.
- **Audit new dependencies** before adding — check download counts, maintenance status, license.

---

## 8. HTTP Security Headers

### Express.js (Node.js backend)

```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.example.com'],
      fontSrc: ["'self'"],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      baseUri: ["'self'"],
    },
  },
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: { policy: 'same-origin' },
  crossOriginResourcePolicy: { policy: 'same-origin' },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  xContentTypeOptions: true, // nosniff
  xFrameOptions: { action: 'deny' },
}));
```

### Required Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS |
| `Content-Security-Policy` | See above | Prevent XSS and injection |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME type sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Control referrer leakage |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Restrict browser features |

---

## 9. CORS Configuration

```typescript
// ❌ DANGEROUS: Allow all origins
app.use(cors({ origin: '*' })); // Never in production!

// ✅ Explicit allowed origins
app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400,
}));
```

### Rules

- **Never use `origin: '*'`** in production.
- **Explicitly list allowed origins**, methods, and headers.
- **Enable `credentials: true`** only when cookies/auth are needed.
- **Set `maxAge`** to cache preflight responses.

---

## 10. Error Handling & Information Leakage

```typescript
// ❌ Leaks internal information
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  res.status(500).json({
    error: err.message,
    stack: err.stack,      // ❌ Exposes internals
    query: req.query,      // ❌ Exposes request data
  });
});

// ✅ Safe error handler
app.use((err: Error, req: Request, res: Response, _next: NextFunction) => {
  const errorId = crypto.randomUUID();
  logger.error('Unhandled error', { errorId, error: err.message, stack: err.stack });

  res.status(500).json({
    error: 'An internal error occurred.',
    errorId,  // For support reference
  });
});
```

### Angular Global Error Handler

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly logger = inject(LoggerService);

  handleError(error: unknown): void {
    // Log full error internally
    this.logger.error('Unhandled error', error);

    // Show user-friendly message — NEVER show stack traces
    // Redirect to error page or show notification
  }
}
```

### Rules

- **Never expose stack traces** in API responses.
- **Never expose file paths, database details, or internal IDs** in error messages.
- **Log full errors server-side** for debugging.
- **Return generic error messages** with a correlation ID.
- **Use different `environment.ts`** configs for dev vs. prod error verbosity.

---

## 11. Sensitive Data in Logs

```typescript
// ❌ Logging sensitive data
logger.info('User login', { email: user.email, password: user.password });
logger.debug('API call', { headers: req.headers }); // May contain Authorization

// ✅ Redact sensitive fields
logger.info('User login', { email: user.email, userId: user.id });
logger.debug('API call', {
  method: req.method,
  url: req.url,
  // Explicitly list safe headers
  userAgent: req.headers['user-agent'],
});

// ✅ Use a redaction utility
function redact<T extends Record<string, unknown>>(
  obj: T,
  sensitiveKeys: string[],
): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(obj).map(([key, value]) =>
      sensitiveKeys.includes(key.toLowerCase())
        ? [key, '***REDACTED***']
        : [key, value],
    ),
  );
}
```

### Sensitive Fields to Never Log

- Passwords, password hashes
- API keys, tokens, secrets
- Credit card numbers, CVVs
- Social security numbers, national IDs
- Full request headers (may contain `Authorization`)
- Session cookies

---

## 12. Rate Limiting

### Express.js

```typescript
import rateLimit from 'express-rate-limit';

// Global rate limit
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                  // 100 requests per window
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many requests, please try again later.' },
}));

// Stricter limit for authentication
app.use('/api/auth', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,  // 5 login attempts per 15 min
  skipSuccessfulRequests: true,
}));
```

### Rules

- **Apply global rate limiting** to all API endpoints.
- **Apply stricter limits** to authentication, password reset, and registration.
- **Return `429 Too Many Requests`** with `Retry-After` header.
- **Use distributed rate limiting** (Redis) in multi-instance deployments.

---

## 13. File Upload Security

```typescript
import multer from 'multer';
import path from 'node:path';

const ALLOWED_MIME_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5 MB

const upload = multer({
  storage: multer.memoryStorage(), // Don't write to disk directly
  limits: {
    fileSize: MAX_FILE_SIZE,
    files: 1,
  },
  fileFilter: (_req, file, cb) => {
    if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) {
      return cb(new Error('Invalid file type'));
    }

    const ext = path.extname(file.originalname).toLowerCase();
    if (!['.jpg', '.jpeg', '.png', '.webp'].includes(ext)) {
      return cb(new Error('Invalid file extension'));
    }

    cb(null, true);
  },
});
```

### Rules

- **Validate MIME type AND file extension** — both can be spoofed, but together are harder to bypass.
- **Limit file size** — reject oversized uploads early.
- **Never use the original filename** — generate a UUID-based name.
- **Store uploads outside the web root** or in cloud storage (S3, Azure Blob).
- **Scan uploaded files** for malware in production.

---

## 14. Angular-Specific Security

### Template Injection

```typescript
// 🚫 NEVER compile user-provided templates
// This allows arbitrary code execution:
const component = this.compiler.compileModuleAsync(userTemplate); // ❌

// 🚫 NEVER use user input in Angular expressions
// In template: {{ userExpression }} where userExpression comes from API — ❌
```

### DomSanitizer

```typescript
// ✅ Only bypass sanitizer when content is fully trusted and immutable
// Always document WHY it's safe
@Pipe({ name: 'safeHtml', standalone: true })
export class SafeHtmlPipe implements PipeTransform {
  private readonly sanitizer = inject(DomSanitizer);

  transform(value: string): SafeHtml {
    // SECURITY: Only used for static admin-authored content
    // stored in our CMS, never for user-generated content.
    return this.sanitizer.bypassSecurityTrustHtml(value);
  }
}
```

### Zone.js & Server-Side Rendering (SSR)

```typescript
// ✅ SSR: Never expose server-side secrets in TransferState
// Only transfer public data
this.transferState.set(
  makeStateKey<User[]>('users'),
  publicUserData, // ❌ Never include tokens, internal IDs, or admin flags
);
```

---

## 15. Cryptography

```typescript
// ✅ Use Web Crypto API (browser) or Node.js crypto module
import { randomBytes, createHash, timingSafeEqual } from 'node:crypto';

// ✅ Secure random token generation
function generateToken(length = 32): string {
  return randomBytes(length).toString('hex');
}

// ✅ Timing-safe comparison (prevents timing attacks)
function safeCompare(a: string, b: string): boolean {
  const bufA = Buffer.from(a, 'utf8');
  const bufB = Buffer.from(b, 'utf8');
  if (bufA.length !== bufB.length) return false;
  return timingSafeEqual(bufA, bufB);
}

// ❌ Never use Math.random() for security-sensitive values
const token = Math.random().toString(36); // ❌ Predictable!

// ❌ Never implement custom crypto
function myEncrypt(data: string, key: string) { ... } // ❌
```

### Password Hashing (Server-Side)

```typescript
import bcrypt from 'bcrypt';

// ✅ Hash with sufficient rounds
const SALT_ROUNDS = 12;
const hash = await bcrypt.hash(password, SALT_ROUNDS);

// ✅ Verify
const isValid = await bcrypt.compare(inputPassword, storedHash);

// 🔜 Prefer argon2 for new projects
import argon2 from 'argon2';
const hash = await argon2.hash(password, { type: argon2.argon2id });
const isValid = await argon2.verify(storedHash, inputPassword);
```

---

## OWASP Top 10 — Quick Reference

| # | Vulnerability | Prevention in JS/TS/Angular |
|---|--------------|----------------------------|
| A01 | Broken Access Control | Route guards, role-based guards, server-side authorization |
| A02 | Cryptographic Failures | Web Crypto API, bcrypt/argon2, HTTPS everywhere |
| A03 | Injection (XSS, SQL, NoSQL) | Angular auto-escaping, parameterized queries, Zod validation |
| A04 | Insecure Design | Threat modeling, ADRs, security reviews |
| A05 | Security Misconfiguration | Helmet headers, CSP, strict CORS, `npm audit` |
| A06 | Vulnerable Components | Dependabot, `npm audit`, lock files, provenance checks |
| A07 | Auth Failures | HTTP-only cookies, short-lived JWTs, token refresh, MFA |
| A08 | Data Integrity Failures | Lock files, `npm ci`, CSP, Subresource Integrity (SRI) |
| A09 | Logging & Monitoring | Structured logging, redaction, error tracking (Sentry) |
| A10 | Server-Side Request Forgery | URL allowlisting, no user-controlled URLs in server fetch |

---

## LLM Agent Directives

When generating or modifying code, the agent MUST:

- Never generate code with `eval()`, `Function()`, or `innerHTML` with user input.
- Never generate code that stores tokens in `localStorage` or `sessionStorage`.
- Never hardcode secrets, API keys, or credentials in source code.
- Never call `bypassSecurityTrust*` without documenting why it's safe.
- Always use parameterized queries — never string interpolation in SQL/NoSQL.
- Always add input validation (Zod, Validators) for every user input path.
- Always include CSRF protection via `withXsrfConfiguration()`.
- Always use `{ credentials: 'include' }` or Angular's `withCredentials` for auth cookies.
- Flag any use of `*` in CORS origin as a **critical security issue**.
- Recommend `helmet` for every Express.js/NestJS application.
- Include security headers in nginx/Docker configurations.
- Add `npm audit` to every CI pipeline configuration.
