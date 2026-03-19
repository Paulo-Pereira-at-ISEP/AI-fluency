---
description: "Testing standards and best practices for JavaScript, TypeScript, and Angular projects using Jest, Angular TestBed, and Playwright."
---

# Testing Standards — JavaScript / TypeScript / Angular

## General Principles

- Every feature must have tests **before** being merged.
- Follow the **Testing Trophy** model: many integration tests, fewer unit tests for pure logic, E2E for critical paths.
- Tests are **documentation** — a reader should understand the behavior from test names alone.
- Tests must be **deterministic** — no flaky tests, no external dependencies, no time-sensitive logic without mocking.
- **Security tests** are mandatory for every endpoint and user input handler.

## Test File Organization

```
src/
├── app/
│   ├── core/
│   │   └── services/
│   │       ├── auth.service.ts
│   │       └── auth.service.spec.ts        # Unit test (co-located)
│   ├── features/
│   │   └── users/
│   │       ├── user-list.component.ts
│   │       ├── user-list.component.spec.ts  # Component test
│   │       └── user.service.spec.ts
│   └── shared/
│       └── utils/
│           ├── validators.ts
│           └── validators.spec.ts
├── tests/
│   ├── integration/                         # Integration tests
│   │   └── user-api.integration.spec.ts
│   └── e2e/                                 # E2E tests (Playwright)
│       ├── login.e2e-spec.ts
│       └── dashboard.e2e-spec.ts
```

- **Unit tests**: co-located with source files as `*.spec.ts`.
- **Integration tests**: in `tests/integration/`.
- **E2E tests**: in `tests/e2e/` using Playwright.

## Jest — Unit Tests

### Test Structure (Arrange-Act-Assert)

```typescript
describe('UserService', () => {
  let service: UserService;
  let httpMock: jest.Mocked<HttpClient>;

  beforeEach(() => {
    httpMock = {
      get: jest.fn(),
      post: jest.fn(),
    } as unknown as jest.Mocked<HttpClient>;

    service = new UserService(httpMock);
  });

  describe('getById', () => {
    it('should return the user when found', async () => {
      // Arrange
      const expectedUser: User = { id: '1', name: 'Alice', email: 'alice@example.com' };
      httpMock.get.mockReturnValue(of(expectedUser));

      // Act
      const result = await firstValueFrom(service.getById('1'));

      // Assert
      expect(result).toEqual(expectedUser);
      expect(httpMock.get).toHaveBeenCalledWith('/api/users/1');
    });

    it('should throw NotFoundError when user does not exist', async () => {
      // Arrange
      httpMock.get.mockReturnValue(
        throwError(() => new HttpErrorResponse({ status: 404 }))
      );

      // Act & Assert
      await expect(firstValueFrom(service.getById('999')))
        .rejects.toThrow(NotFoundError);
    });
  });
});
```

### Naming Convention

```typescript
// Pattern: "should <expected behavior> when <condition>"
it('should return empty array when no users match the filter', () => { ... });
it('should throw ValidationError when email format is invalid', () => { ... });
it('should emit userSelected event when card is clicked', () => { ... });
```

### Mocking

```typescript
// ✅ jest.fn() for simple mocks
const mockLogger = { error: jest.fn(), warn: jest.fn(), info: jest.fn() };

// ✅ jest.spyOn for partial mocks
jest.spyOn(Date, 'now').mockReturnValue(1710000000000);

// ✅ jest.mock for module mocking
jest.mock('@core/services/analytics.service', () => ({
  AnalyticsService: jest.fn().mockImplementation(() => ({
    track: jest.fn(),
  })),
}));

// ✅ Typed mocks with jest.Mocked
const mockRepo = jest.mocked(userRepository);
mockRepo.findById.mockResolvedValue(testUser);

// ❌ Never mock what you don't own — write integration tests instead
// ❌ Never mock implementation details — mock interfaces/contracts
```

### Parameterized Tests

```typescript
describe('validateEmail', () => {
  it.each([
    ['user@example.com', true],
    ['user@sub.domain.com', true],
    ['invalid', false],
    ['@missing-local.com', false],
    ['user@', false],
    ['', false],
  ])('should return %s for input "%s"', (input, expected) => {
    expect(validateEmail(input)).toBe(expected);
  });
});

// Object syntax for complex cases
it.each([
  { input: { age: -1 }, error: 'Age must be non-negative' },
  { input: { age: 200 }, error: 'Age must be below 150' },
  { input: { age: 25 }, error: null },
])('should validate age=$input.age → error=$error', ({ input, error }) => {
  const result = validateUser(input);
  if (error) {
    expect(result.errors).toContain(error);
  } else {
    expect(result.isValid).toBe(true);
  }
});
```

## Angular — Component Testing

### TestBed with Standalone Components

```typescript
describe('UserCardComponent', () => {
  let fixture: ComponentFixture<UserCardComponent>;
  let component: UserCardComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserCardComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
  });

  it('should display the user name', () => {
    // Set signal-based input
    fixture.componentRef.setInput('user', {
      id: '1',
      firstName: 'Alice',
      lastName: 'Smith',
    });
    fixture.detectChanges();

    const nameEl = fixture.nativeElement.querySelector('[data-testid="user-name"]');
    expect(nameEl.textContent).toContain('Alice Smith');
  });

  it('should emit userSelected when card is clicked', () => {
    const user: User = { id: '1', firstName: 'Alice', lastName: 'Smith' };
    fixture.componentRef.setInput('user', user);
    fixture.detectChanges();

    const emitSpy = jest.spyOn(component.userSelected, 'emit');
    const card = fixture.nativeElement.querySelector('[data-testid="user-card"]');
    card.click();

    expect(emitSpy).toHaveBeenCalledWith(user);
  });
});
```

### Testing Services with HttpClientTestingModule

```typescript
describe('UserApiService', () => {
  let service: UserApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        UserApiService,
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: API_URL, useValue: 'https://api.test.com' },
      ],
    });

    service = TestBed.inject(UserApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Ensure no outstanding requests
  });

  it('should fetch users from the API', () => {
    const mockUsers: User[] = [{ id: '1', name: 'Alice' }];

    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('https://api.test.com/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});
```

### Component Harnesses

```typescript
// ✅ Use Angular CDK component harnesses for mat-* components
import { MatButtonHarness } from '@angular/material/button/testing';

it('should disable submit button when form is invalid', async () => {
  const loader = TestbedHarnessEnvironment.loader(fixture);
  const submitBtn = await loader.getHarness(
    MatButtonHarness.with({ selector: '[data-testid="submit-btn"]' })
  );

  expect(await submitBtn.isDisabled()).toBe(true);
});
```

## Playwright — E2E Tests

### Page Object Model

```typescript
// pages/login.page.ts
export class LoginPage {
  constructor(private readonly page: Page) {}

  async goto(): Promise<void> {
    await this.page.goto('/login');
  }

  async login(email: string, password: string): Promise<void> {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign In' }).click();
  }

  async getErrorMessage(): Promise<string | null> {
    return this.page.getByTestId('error-message').textContent();
  }
}

// tests/e2e/login.e2e-spec.ts
test.describe('Login', () => {
  let loginPage: LoginPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    await loginPage.goto();
  });

  test('should redirect to dashboard after successful login', async ({ page }) => {
    await loginPage.login('admin@example.com', 'validPassword123');
    await expect(page).toHaveURL('/dashboard');
  });

  test('should show error message for invalid credentials', async () => {
    await loginPage.login('admin@example.com', 'wrongPassword');
    const error = await loginPage.getErrorMessage();
    expect(error).toContain('Invalid credentials');
  });
});
```

### Best Practices

```typescript
// ✅ Use data-testid for selectors (resilient to UI changes)
await page.getByTestId('submit-button').click();

// ✅ Use role-based selectors for accessibility
await page.getByRole('button', { name: 'Submit' }).click();

// ✅ Use web-first assertions (auto-retry)
await expect(page.getByTestId('user-list')).toBeVisible();
await expect(page.getByTestId('user-count')).toHaveText('42');

// ❌ Never use CSS selectors tied to styling
await page.locator('.btn-primary.mt-3').click();

// ❌ Never use arbitrary waits
await page.waitForTimeout(3000);
```

## Security Tests

> Security tests are **mandatory** for every API endpoint and user input handler. Tag them with a descriptive category.

### XSS Prevention

```typescript
describe('XSS Prevention', () => {
  const xssPayloads = [
    '<script>alert("xss")</script>',
    '<img src=x onerror=alert(1)>',
    'javascript:alert(1)',
    '"><script>alert(document.cookie)</script>',
    "'; DROP TABLE users; --",
  ];

  it.each(xssPayloads)(
    'should sanitize XSS payload: %s',
    (payload) => {
      const result = sanitizeInput(payload);
      expect(result).not.toContain('<script');
      expect(result).not.toContain('onerror');
      expect(result).not.toContain('javascript:');
    },
  );
});
```

### Authentication & Authorization

```typescript
describe('Auth Guards', () => {
  it('should redirect unauthenticated users to login', async () => {
    // Arrange: no token in storage
    localStorage.clear();

    // Act
    const result = await TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
    );

    // Assert
    expect(result).toEqual(expect.objectContaining({
      path: '/login',
    }));
  });

  it('should deny access to admin routes for non-admin users', async () => {
    // Arrange
    mockAuthService.currentUser.mockReturnValue(signal({ role: 'viewer' }));

    // Act & Assert
    const result = await TestBed.runInInjectionContext(() =>
      adminGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
    );
    expect(result).toBe(false);
  });
});
```

### Input Validation

```typescript
describe('Input Validation', () => {
  it.each([
    ['../../../etc/passwd', 'path traversal'],
    ['..\\..\\windows\\system32', 'windows path traversal'],
    ['/etc/shadow', 'absolute path'],
  ])('should reject path traversal attempt: %s (%s)', (input) => {
    expect(() => validateFilePath(input)).toThrow();
  });

  it('should reject payloads exceeding maximum size', () => {
    const oversizedPayload = 'x'.repeat(10_001);
    expect(() => validateInput(oversizedPayload)).toThrow('Input exceeds maximum length');
  });
});
```

### E2E Security Tests

```typescript
test.describe('Security - E2E', () => {
  test('should include security headers in responses', async ({ page }) => {
    const response = await page.goto('/');
    const headers = response!.headers();

    expect(headers['x-content-type-options']).toBe('nosniff');
    expect(headers['x-frame-options']).toBe('DENY');
    expect(headers['strict-transport-security']).toBeDefined();
    expect(headers['content-security-policy']).toBeDefined();
  });

  test('should not expose stack traces in error responses', async ({ request }) => {
    const response = await request.get('/api/nonexistent');
    const body = await response.json();

    expect(body).not.toHaveProperty('stack');
    expect(body).not.toHaveProperty('stackTrace');
    expect(JSON.stringify(body)).not.toContain('at ');
  });
});
```

## Code Coverage

### Configuration (jest.config.ts)

```typescript
export default {
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.e2e-spec.ts',
    '!src/**/index.ts',
    '!src/main.ts',
    '!src/environments/**',
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

### Targets

| Metric | Minimum | Recommended |
|--------|---------|-------------|
| Line coverage | 80% | 90%+ |
| Branch coverage | 80% | 85%+ |
| Function coverage | 80% | 90%+ |
| Security-critical code | 95% | 100% |

## Test Data

```typescript
// ✅ Use builder/factory pattern for test data
function createTestUser(overrides: Partial<User> = {}): User {
  return {
    id: 'test-user-1',
    firstName: 'Alice',
    lastName: 'Smith',
    email: 'alice@example.com',
    role: 'viewer',
    createdAt: '2026-01-01T00:00:00Z',
    ...overrides,
  };
}

// Usage
const admin = createTestUser({ role: 'admin', firstName: 'Bob' });
const viewer = createTestUser(); // defaults

// ✅ Use constants for magic values
const VALID_EMAIL = 'test@example.com';
const INVALID_UUID = 'not-a-uuid';
const EXPIRED_TOKEN = 'eyJ...expired';
```

## npm Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --reporters=default --reporters=jest-junit",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:security": "jest --testPathPattern=security"
  }
}
```

## LLM Agent Directives

When generating or modifying tests, the agent MUST:

- Generate tests in the same `describe`/`it` structure shown above.
- Use Arrange-Act-Assert pattern with clear section comments.
- Follow naming convention: `should <expected> when <condition>`.
- Use `jest.fn()` and `jest.spyOn()` for mocking — never mock implementation details.
- Use `it.each` for parameterized tests with 3+ cases.
- Use `data-testid` attributes for E2E selectors.
- Include security tests for every endpoint or input handler.
- Never generate tests that depend on execution order.
- Never generate tests with hardcoded timeouts or `sleep()`.
- Use test data builders instead of inline object literals.
- Verify `httpMock.verify()` is called in `afterEach` for HTTP tests.
