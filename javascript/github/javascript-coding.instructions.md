---
description: "Coding conventions and best practices for JavaScript, TypeScript, and Angular development."
---

# JavaScript / TypeScript / Angular — Coding Conventions

## General Principles

- **TypeScript is mandatory** for all application code. Enable `strict: true` in every `tsconfig.json`.
- **JavaScript** is only acceptable for tooling configuration files (ESLint, Prettier, etc.).
- Prefer **immutability** — use `const` by default, `let` only when reassignment is necessary, never `var`.
- Prefer **pure functions** and declarative patterns over imperative, mutation-heavy code.
- Keep functions **small and focused** — a function should do one thing.
- **No `any`** — use `unknown` when the type truly cannot be determined, then narrow with type guards.

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Variables, functions | `camelCase` | `userName`, `getUserById()` |
| Classes, interfaces, types, enums | `PascalCase` | `UserService`, `HttpInterceptor` |
| Constants (true constants) | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `API_BASE_URL` |
| Enum members | `PascalCase` | `UserRole.Admin` |
| Angular components | `PascalCase` class, `kebab-case` selector | `UserCardComponent`, `app-user-card` |
| Angular services | `PascalCase` with `Service` suffix | `AuthService`, `UserApiService` |
| Interfaces | `PascalCase`, **no** `I` prefix | `User`, `ApiResponse<T>` (not `IUser`) |
| Type aliases | `PascalCase` | `UserId`, `HttpMethod` |
| File names | `kebab-case` | `user-card.component.ts`, `auth.service.ts` |
| Test files | `*.spec.ts` | `user.service.spec.ts` |
| E2E test files | `*.e2e-spec.ts` | `login.e2e-spec.ts` |
| Private members | No prefix (use `private` keyword) | `private readonly userRepo` |

## TypeScript Strict Typing

### tsconfig.json — Mandatory Settings

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Type Annotations

```typescript
// ✅ Explicit return types on public functions
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// ✅ Use generics for reusable code
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

// ❌ Never use `any`
function process(data: any) { ... }

// ✅ Use `unknown` + type narrowing
function process(data: unknown): void {
  if (isUser(data)) {
    console.log(data.name);
  }
}
```

### Utility Types

```typescript
// Prefer built-in utility types
type UserUpdate = Partial<User>;
type RequiredUser = Required<User>;
type ReadonlyUser = Readonly<User>;
type UserName = Pick<User, 'firstName' | 'lastName'>;
type UserWithoutPassword = Omit<User, 'password'>;
type ApiResponse<T> = { data: T; status: number; message: string };

// Discriminated unions for state management
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```

### Type Guards

```typescript
// Custom type guard
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// Use `satisfies` for type validation without widening
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
} satisfies AppConfig;
```

### Enums vs Union Types

```typescript
// ✅ Prefer const enums or string literal unions for simple cases
type UserRole = 'admin' | 'editor' | 'viewer';

// ✅ Use const enum for numeric values (erased at compile time)
const enum HttpStatus {
  Ok = 200,
  NotFound = 404,
  ServerError = 500,
}

// ✅ Use regular enum only when runtime access to keys/values is needed
enum LogLevel {
  Debug = 'DEBUG',
  Info = 'INFO',
  Warn = 'WARN',
  Error = 'ERROR',
}
```

## Angular — Component Patterns

### Standalone Components (Angular 17+)

```typescript
// ✅ Always standalone: true (default since Angular 17)
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './user-card.component.html',
  styleUrl: './user-card.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  // Signal-based inputs (Angular 17.1+)
  readonly user = input.required<User>();
  readonly showActions = input(false);

  // Signal-based outputs (Angular 17.3+)
  readonly userSelected = output<User>();

  // Computed signals
  readonly displayName = computed(() =>
    `${this.user().firstName} ${this.user().lastName}`
  );

  onSelect(): void {
    this.userSelected.emit(this.user());
  }
}
```

### Signals & Reactive State

```typescript
// ✅ Use signals for component state
@Component({ ... })
export class DashboardComponent {
  private readonly userService = inject(UserService);

  readonly searchTerm = signal('');
  readonly users = signal<User[]>([]);
  readonly isLoading = signal(false);

  readonly filteredUsers = computed(() =>
    this.users().filter(u =>
      u.name.toLowerCase().includes(this.searchTerm().toLowerCase())
    )
  );

  readonly userCount = computed(() => this.filteredUsers().length);

  async loadUsers(): Promise<void> {
    this.isLoading.set(true);
    try {
      const data = await firstValueFrom(this.userService.getAll());
      this.users.set(data);
    } finally {
      this.isLoading.set(false);
    }
  }
}
```

### Dependency Injection

```typescript
// ✅ Use inject() function (preferred over constructor injection)
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly config = inject(APP_CONFIG);

  getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.config.apiUrl}/users/${id}`);
  }
}

// ✅ Use InjectionToken for configuration values
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// ✅ Provide in route or component, not in module
export const appRoutes: Routes = [
  {
    path: 'admin',
    providers: [AdminService],
    children: [...]
  },
];
```

### Template Syntax (Angular 17+)

```html
<!-- ✅ New control flow syntax (Angular 17+) -->
@if (isLoading()) {
  <app-spinner />
} @else if (error()) {
  <app-error-message [error]="error()" />
} @else {
  @for (user of users(); track user.id) {
    <app-user-card [user]="user" (userSelected)="onSelect($event)" />
  } @empty {
    <p>No users found.</p>
  }
}

@switch (status()) {
  @case ('active') { <span class="badge-active">Active</span> }
  @case ('inactive') { <span class="badge-inactive">Inactive</span> }
  @default { <span class="badge-unknown">Unknown</span> }
}

<!-- ✅ Defer loading for heavy components -->
@defer (on viewport) {
  <app-heavy-chart [data]="chartData()" />
} @placeholder {
  <div class="chart-placeholder">Loading chart...</div>
} @loading (minimum 300ms) {
  <app-spinner />
}
```

### Angular Services & HTTP

```typescript
// ✅ Typed HTTP responses with error handling
@Injectable({ providedIn: 'root' })
export class UserApiService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = inject(API_URL);

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`).pipe(
      retry({ count: 2, delay: 1000 }),
      catchError(this.handleError),
    );
  }

  createUser(dto: CreateUserDto): Observable<User> {
    return this.http.post<User>(`${this.apiUrl}/users`, dto);
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    const message = error.error?.message ?? 'An unexpected error occurred';
    console.error('API Error:', message);
    return throwError(() => new Error(message));
  }
}
```

## RxJS Best Practices

```typescript
// ✅ Use higher-order mapping operators
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.userService.search(term)),
  takeUntilDestroyed(this.destroyRef),
).subscribe(users => this.users.set(users));

// ✅ Use takeUntilDestroyed() for automatic cleanup (Angular 16+)
export class MyComponent {
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.dataService.getData().pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(data => this.process(data));
  }
}

// ❌ Never subscribe without cleanup
// ❌ Never nest .subscribe() calls
// ❌ Avoid .toPromise() — use firstValueFrom() or lastValueFrom()
```

## Async Patterns

```typescript
// ✅ async/await for single-value operations
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new HttpError(response.status, response.statusText);
  }
  return response.json() as Promise<User>;
}

// ✅ Use Promise.all for parallel operations
const [users, roles, permissions] = await Promise.all([
  fetchUsers(),
  fetchRoles(),
  fetchPermissions(),
]);

// ✅ Use Promise.allSettled when partial failure is acceptable
const results = await Promise.allSettled(urls.map(url => fetch(url)));
const successful = results
  .filter((r): r is PromiseFulfilledResult<Response> => r.status === 'fulfilled')
  .map(r => r.value);
```

## Error Handling

```typescript
// ✅ Custom error classes
export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class NotFoundError extends AppError {
  constructor(entity: string, id: string) {
    super(`${entity} with id '${id}' not found`, 'NOT_FOUND', 404);
    this.name = 'NotFoundError';
  }
}

// ✅ Angular error handler
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly logger = inject(LoggerService);

  handleError(error: unknown): void {
    if (error instanceof HttpErrorResponse) {
      this.logger.error('HTTP error', { status: error.status, url: error.url });
    } else if (error instanceof Error) {
      this.logger.error('Application error', { message: error.message, stack: error.stack });
    } else {
      this.logger.error('Unknown error', { error });
    }
  }
}
```

## Imports & Module Organization

```typescript
// ✅ Order: Angular → third-party → project (with blank lines between groups)
import { Component, inject, signal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';

import { TranslateModule } from '@ngx-translate/core';

import { UserService } from '@core/services/user.service';
import { User } from '@shared/models/user.model';
import { SpinnerComponent } from '@shared/components/spinner/spinner.component';
```

## Formatting & Style

- **Prettier** handles all formatting — no manual style debates.
- **Semicolons**: always.
- **Trailing commas**: always (`es5` setting in Prettier).
- **Single quotes** for strings.
- **2-space indentation**.
- **Max line length**: 100 characters.
- Use **template literals** for string interpolation, never concatenation.

## Security

> See `security.instructions.md` for comprehensive guidelines.

Essential rules for every TypeScript/Angular file:

1. **Never use `innerHTML` binding** without sanitization — prefer Angular text interpolation `{{ }}`.
2. **Never call `bypassSecurityTrustHtml/Url/Script/ResourceUrl`** without explicit security review.
3. **Never use `eval()`**, `Function()` constructor, or `new Function()`.
4. **Never hardcode** API keys, tokens, or secrets in source code.
5. **Always validate** user input both client-side and server-side.
6. **Always use** `HttpClient` with typed responses — never use `fetch()` without proper error handling.
7. **Always use** Angular's built-in CSRF protection (`HttpClientXsrfModule` or `withXsrfConfiguration()`).
8. **Always run** `npm audit` before every release.
9. **Never disable** TypeScript strict checks to suppress warnings.
10. **Sanitize** all dynamic content rendered in templates.

## LLM Agent Directives

When generating or modifying TypeScript/Angular code, the agent MUST:

- Generate TypeScript, never plain JavaScript, for application code.
- Enable and respect `strict: true` — never emit code that requires disabling strict checks.
- Use Angular standalone components with signals, never NgModules for new code.
- Use `inject()` function for dependency injection, not constructor parameters.
- Use the new `@if`/`@for`/`@switch` control flow syntax, never `*ngIf`/`*ngFor`.
- Add proper types to every function parameter and return value.
- Follow the import ordering convention.
- Include `ChangeDetectionStrategy.OnPush` on every component.
- Use `trackBy` (via `track` in `@for`) on every list iteration.
- Validate security implications of every generated code block against `security.instructions.md`.
