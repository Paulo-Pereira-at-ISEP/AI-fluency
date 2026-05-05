---
description: "Comprehensive security guidelines for TypeScript Node.js projects. Covers OWASP Top 10 for APIs, input validation, authentication, secrets management, and dependency auditing."
---

# Security Guidelines — TypeScript / Node.js

## Core Principles

1. **Security is not optional** — every feature must consider security from design to deployment.
2. **Defence in depth** — multiple layers of protection; never rely on a single control.
3. **Least privilege** — grant minimum necessary access to users, services, and code.
4. **Fail securely** — errors must not expose sensitive data or bypass security controls.
5. **Validate everything at the boundary** — treat all external input as untrusted until validated.

---

## 1. Input Validation & Sanitization

All external inputs — HTTP bodies, query params, headers, env vars, config files, CLI args — **must** be validated with Zod before use.

```typescript
import { z } from 'zod';

// ✅ Strict schema: whitelist allowed fields, reject unknown
const CreateUserSchema = z.object({
  name: z.string().min(2).max(100).trim(),
  email: z.string().email().toLowerCase(),
  age: z.number().int().min(0).max(150),
  role: z.enum(['admin', 'editor', 'viewer']),
}).strict(); // rejects unknown properties

type CreateUserDto = z.infer<typeof CreateUserSchema>;

// ✅ Use safeParse at HTTP boundary, throw AppError on failure
function parseBody<T>(schema: z.ZodSchema<T>, body: unknown): T {
  const result = schema.safeParse(body);
  if (!result.success) {
    throw new ValidationError('Request validation failed', result.error.flatten());
  }
  return result.data;
}
```

### Rules

- **Always validate on the server** — client-side validation is UX only.
- **Whitelist allowed values** — use `z.enum()`, `z.literal()`, min/max constraints.
- **Reject unknown fields** — use `.strict()` on Zod objects.
- **Sanitize strings** — trim whitespace; normalize emails to lowercase.
- **Validate types, length, range, and format** for every field.
- **Never trust `Content-Type` alone** — always parse and validate the body.

---

## 2. SQL Injection Prevention

```typescript
// ❌ SQL injection vulnerability
const query = `SELECT * FROM users WHERE id = '${req.params.id}'`;

// ✅ Parameterized query (pg)
const { rows } = await pool.query(
  'SELECT id, name, email FROM users WHERE id = $1',
  [userId], // userId is a validated string from Zod
);

// ✅ Prisma ORM (safe by default — uses prepared statements)
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: { id: true, name: true, email: true }, // never select * in code
});

// ❌ NoSQL injection (MongoDB)
// Dangerous if req.body.name is { $gt: '' }
const user = await db.collection('users').findOne({ name: req.body.name });

// ✅ Always validate input types before querying
const { name } = parseBody(SearchSchema, req.body); // name is guaranteed a string
const user = await db.collection('users').findOne({ name });
```

### Rules

- **Never interpolate user input** into SQL or NoSQL query strings.
- **Use parameterized queries** or ORM methods exclusively.
- **Validate and type-check** before querying — Zod ensures strings are strings.
- **Limit selected columns** — avoid `SELECT *`; only return needed fields.

---

## 3. Authentication & JWT

```typescript
import jwt from 'jsonwebtoken';
import { z } from 'zod';

const JwtPayloadSchema = z.object({
  sub: z.string().uuid(),
  role: z.enum(['admin', 'editor', 'viewer']),
  iat: z.number(),
  exp: z.number(),
});

type JwtPayload = z.infer<typeof JwtPayloadSchema>;

// ✅ Always verify and validate JWT payload
function verifyToken(token: string): JwtPayload {
  const secret = env.JWT_SECRET; // validated at startup via Zod
  let decoded: unknown;
  try {
    decoded = jwt.verify(token, secret, { algorithms: ['HS256'] });
  } catch {
    throw new UnauthorizedError('Invalid or expired token');
  }
  const result = JwtPayloadSchema.safeParse(decoded);
  if (!result.success) throw new UnauthorizedError('Malformed token payload');
  return result.data;
}

// ✅ Extract token from Authorization header only — never from query params
function extractBearerToken(req: Request): string {
  const auth = req.headers.authorization;
  if (!auth?.startsWith('Bearer ')) throw new UnauthorizedError('Missing authorization header');
  return auth.slice(7);
}
```

### Token Storage Rules

- **Access tokens**: store in memory (JavaScript variable) — never in `localStorage`.
- **Refresh tokens**: store in HTTP-only, Secure, SameSite=Strict cookies — set by server.
- **Never log tokens** — mask or omit from all log output.
- **Use short-lived access tokens** (15 min) with longer-lived refresh tokens (7 days).
- **Validate `alg` header** — never accept `alg: none`.

---

## 4. Authorization (RBAC)

```typescript
// ✅ Typed permission check middleware
function requireRole(...allowedRoles: UserRole[]): RequestHandler {
  return (req, res, next) => {
    const user = req.user; // set by auth middleware
    if (!user) return next(new UnauthorizedError('Authentication required'));
    if (!allowedRoles.includes(user.role)) {
      return next(new ForbiddenError('Insufficient permissions'));
    }
    next();
  };
}

// Usage in router
router.delete('/users/:id', authenticate, requireRole('admin'), deleteUserHandler);

// ✅ Resource-level authorization — check ownership, not just role
async function deleteUser(requesterId: string, requesterRole: UserRole, targetId: string): Promise<void> {
  if (requesterRole !== 'admin' && requesterId !== targetId) {
    throw new ForbiddenError('Cannot delete another user account');
  }
  await userRepository.delete(targetId);
}
```

---

## 5. Secrets Management

```typescript
// 🚫 NEVER hardcode secrets
const API_KEY = 'sk-abc123def456';                    // ❌
const DB_URI = 'postgresql://admin:pass@localhost/db'; // ❌

// ✅ Validate all secrets at application startup with Zod
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  REDIS_URL: z.string().url().optional(),
});

// Fails at startup if any required variable is missing or invalid
export const env = EnvSchema.parse(process.env);
```

### .env Rules

```gitignore
# .gitignore — secrets must never be committed
.env
.env.local
.env.*.local
*.pem
*.key
```

- **Use `.env.example`** with placeholder values (committed) to document required variables.
- **Never commit `.env`** files.
- **Use secrets managers in production** — AWS Secrets Manager, Azure Key Vault, HashiCorp Vault.
- **Rotate secrets** regularly and immediately after suspected compromise.
- **Add `detect-secrets`** as a pre-commit hook to catch accidental secret leaks.

---

## 6. HTTP Security Headers

```typescript
import helmet from 'helmet';

// ✅ Apply Helmet middleware as the first middleware in Express
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
      baseUri: ["'self'"],
      formAction: ["'self'"],
    },
  },
  hsts: { maxAge: 31_536_000, includeSubDomains: true, preload: true },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));

// ✅ CORS — explicit allowlist, never '*' in production
import cors from 'cors';
const allowedOrigins = env.ALLOWED_ORIGINS.split(','); // validated list

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('CORS policy violation'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
}));
```

---

## 7. Rate Limiting & DoS Prevention

```typescript
import rateLimit from 'express-rate-limit';
import slowDown from 'express-slow-down';

// ✅ Global rate limit
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  message: { error: { code: 'RATE_LIMIT_EXCEEDED', message: 'Too many requests' } },
});

// ✅ Stricter limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { error: { code: 'RATE_LIMIT_EXCEEDED', message: 'Too many login attempts' } },
});

app.use(globalLimiter);
app.use('/api/auth', authLimiter);
```

---

## 8. Dependency Auditing

```bash
# ✅ Run on every CI build — fail on high/critical vulnerabilities
npm audit --audit-level=high

# ✅ Auto-fix patch and minor vulnerabilities
npm audit fix
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 10
    groups:
      dev-dependencies:
        dependency-type: development
```

### Rules

- **Run `npm audit`** on every CI pipeline; fail on `high` or `critical` severity.
- **Enable Dependabot** for automated dependency updates.
- **Pin major versions** of critical dependencies; allow minor/patch auto-updates.
- **Review transitive dependencies** — run `npm audit` even if direct deps are clean.

---

## 9. Logging Security

```typescript
// ✅ Never log sensitive data
logger.info('User login attempt', {
  email: maskEmail(user.email),     // mask@example.com
  userId: user.id,
  ip: req.ip,
  // ❌ Never log: password, token, creditCard, ssn, secret
});

function maskEmail(email: string): string {
  const [local, domain] = email.split('@');
  return `${local!.slice(0, 2)}***@${domain}`;
}

// ✅ Structured logging — never log raw Error objects to avoid stack trace leaks in prod
logger.error('Database query failed', {
  errorCode: err instanceof AppError ? err.code : 'UNKNOWN',
  message: err instanceof Error ? err.message : 'Unknown error',
  // No stack trace in production logs visible to users
});
```

### Rules

- **Never log**: passwords, tokens, API keys, credit card numbers, PII in production.
- **Use structured logging** (JSON) — avoids unintentional data leaks via string interpolation.
- **Never return stack traces** in API error responses in production.
- **Sanitize log data** — use masking utilities for emails, phone numbers.

---

## 10. Insecure Randomness

```typescript
// ❌ Math.random() is predictable — never use for security-sensitive values
const token = Math.random().toString(36); // ❌

// ✅ Use Node.js crypto module for cryptographically secure random values
import { randomBytes, createHash } from 'node:crypto';

const secureToken = randomBytes(32).toString('hex');        // 256-bit token
const sessionId = randomBytes(16).toString('base64url');    // 128-bit session ID

// ✅ Secure comparison to prevent timing attacks
import { timingSafeEqual } from 'node:crypto';

function safeCompare(a: string, b: string): boolean {
  const bufA = Buffer.from(a);
  const bufB = Buffer.from(b);
  if (bufA.length !== bufB.length) return false;
  return timingSafeEqual(bufA, bufB);
}
```

---

## 11. Path Traversal Prevention

```typescript
import path from 'node:path';

// ❌ Vulnerable to path traversal
const filePath = path.join('/uploads', req.params.filename); // ../../../../etc/passwd

// ✅ Resolve and verify the path stays within the allowed directory
const BASE_DIR = path.resolve('/uploads');

function resolveSafeFilePath(filename: string): string {
  const resolved = path.resolve(BASE_DIR, filename);
  if (!resolved.startsWith(BASE_DIR + path.sep)) {
    throw new ForbiddenError('Path traversal attempt detected');
  }
  return resolved;
}
```

---

## 12. Security Checklist per Feature

Before merging any feature, verify:

- [ ] All inputs validated with Zod at the entry point.
- [ ] No SQL/NoSQL string interpolation — parameterized queries or ORM used.
- [ ] Auth middleware applied to all protected routes.
- [ ] RBAC checks enforce both role and resource ownership.
- [ ] No secrets in source code or committed `.env` files.
- [ ] Helmet and CORS configured with explicit allowlists.
- [ ] Rate limiting applied to public endpoints.
- [ ] No sensitive data in logs or API error responses.
- [ ] Cryptographic operations use `node:crypto`, not `Math.random()`.
- [ ] `npm audit` passes with no high/critical findings.
