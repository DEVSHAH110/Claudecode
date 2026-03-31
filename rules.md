# rules.md — Framework Rules, Standards & Conventions

## Overview

These rules are **mandatory** for all contributors (human and AI). They ensure the framework remains maintainable, reliable, and scalable. No exceptions without explicit approval from the Framework Architect.

---

## 🔴 Non-Negotiable Rules (Never Violate)

### R1 — No Hard Waits
```typescript
// ❌ NEVER do this
await page.waitForTimeout(3000);
await new Promise(resolve => setTimeout(resolve, 2000));

// ✅ Always do this
await page.waitForLoadState('networkidle');
await page.waitForSelector('[data-testid="modal"]');
await expect(locator).toBeVisible();
```

### R2 — No Raw Locators in Test Files
All UI interactions must go through **Page Object Model** classes.
```typescript
// ❌ NEVER do this in a test file
await page.click('.login-btn');
await page.fill('#email', 'test@example.com');

// ✅ Always do this in a test file
const loginPage = new LoginPage(page);
await loginPage.login('test@example.com', 'password');
```

### R3 — No Hardcoded Credentials or Sensitive Data
```typescript
// ❌ NEVER hardcode secrets
await loginPage.login('admin@company.com', 'SuperSecret123!');

// ✅ Always use environment variables
await loginPage.login(process.env.TEST_EMAIL!, process.env.TEST_PASSWORD!);
```

### R4 — Every Test Must Be Independent
- Tests must not depend on execution order.
- Tests must not share mutable state unless explicitly isolated.
- Each test must clean up its own data.
```typescript
// ❌ Bad: test2 depends on test1 having run first
test('test1 creates user', ...);
test('test2 uses user from test1', ...); // Will fail if run alone

// ✅ Good: each test creates its own data
test.beforeEach(async ({ request }) => {
  // Create fresh data for this test
});
```

### R5 — No `test.only()` or `test.skip()` Committed to Main Branch
These are acceptable locally during development only. Remove before committing.
```typescript
// ❌ NEVER commit these
test.only('focused test', ...);
test.skip('skipped test', ...);
```

### R6 — Always Use TypeScript Strict Mode
All files must be `.ts` or `.tsx`. No `.js` files in `tests/`, `pages/`, `fixtures/`, or `helpers/`.

---

## 🟡 Required Conventions

### Selector Priority Order
Use selectors in this order of preference:
```
1. data-testid attribute    → [data-testid="login-btn"]    ← BEST
2. ARIA role + name         → getByRole('button', { name: 'Submit' })
3. Accessible label/text    → getByLabel('Email address')
4. Visible text             → getByText('Sign in')
5. CSS class (last resort)  → .login-button                ← AVOID
```

Never use:
- XPath (fragile, unreadable)
- Auto-generated IDs (`#field_12345`)
- Positional selectors (`nth-child(3)`)

---

### File Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Test files | `*.spec.ts` | `login.spec.ts` |
| Page Objects | `*Page.ts` (PascalCase) | `LoginPage.ts` |
| Fixtures | `*.fixture.ts` | `auth.fixture.ts` |
| Helpers | `*Helper.ts` or `*.utils.ts` | `apiHelper.ts` |
| Test data | `*.data.ts` or `*.json` | `users.data.ts` |
| Config files | `*.config.ts` | `playwright.config.ts` |

---

### Test File Structure

```typescript
// Standard test file structure
import { test, expect } from '../fixtures';  // Always import from fixtures, not @playwright/test
import { LoginPage } from '../pages/LoginPage';

test.describe('Feature: [Feature Name]', () => {
  
  // Declare page objects
  let loginPage: LoginPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    await loginPage.navigate();
  });

  test('[TC-001] should [expected behavior] when [condition]', async ({ page }) => {
    // Arrange
    const email = process.env.TEST_EMAIL!;
    
    // Act
    await loginPage.login(email, process.env.TEST_PASSWORD!);
    
    // Assert
    await expect(page).toHaveURL('/dashboard');
  });

});
```

---

### Page Object Structure

```typescript
// Standard Page Object Model structure
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  
  // Locators — defined as class properties
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByTestId('email-input');
    this.passwordInput = page.getByTestId('password-input');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByTestId('error-message');
  }

  // Navigation
  async navigate() {
    await this.page.goto('/login');
    await this.page.waitForLoadState('domcontentloaded');
  }

  // Actions — compound steps
  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  // Assertions — verify page state
  async expectErrorMessage(message: string) {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toHaveText(message);
  }
}
```

---

### Test Naming Convention

Format: `[TC-XXX] should [expected outcome] when [condition]`

```typescript
test('[TC-001] should redirect to dashboard when valid credentials provided', ...);
test('[TC-002] should display error when invalid password entered', ...);
test('[TC-003] should lock account after 5 failed login attempts', ...);
```

---

### Assertion Rules

```typescript
// ✅ Use Playwright's built-in assertions (auto-retry)
await expect(locator).toBeVisible();
await expect(locator).toHaveText('Welcome');
await expect(page).toHaveURL('/dashboard');
await expect(locator).toBeEnabled();

// ❌ Avoid raw boolean assertions (no auto-retry = flaky)
expect(await locator.isVisible()).toBe(true);
expect(await page.url()).toBe('https://example.com/dashboard');
```

---

### API Test Rules

```typescript
// Always validate both status code AND response body
test('POST /api/login returns token', async ({ request }) => {
  const response = await request.post('/api/login', {
    data: { email: process.env.TEST_EMAIL, password: process.env.TEST_PASSWORD }
  });
  
  expect(response.status()).toBe(200);             // Status check
  
  const body = await response.json();
  expect(body).toHaveProperty('token');            // Schema check
  expect(typeof body.token).toBe('string');        // Type check
  expect(body.token.length).toBeGreaterThan(0);   // Value check
});
```

---

### Environment Variable Rules

All environment variables must:
1. Be declared in `.env.example` with placeholder values
2. Be documented in `config/README.md`
3. Use the `!` non-null assertion when used in TypeScript (`process.env.VAR!`)
4. Be validated at startup in `globalSetup.ts`

```typescript
// helpers/envValidator.ts
export function validateEnv() {
  const required = ['BASE_URL', 'TEST_EMAIL', 'TEST_PASSWORD', 'API_KEY'];
  const missing = required.filter(key => !process.env[key]);
  if (missing.length > 0) {
    throw new Error(`Missing required env variables: ${missing.join(', ')}`);
  }
}
```

---

### Tagging Convention

```typescript
// Apply tags in the test title
test('Login works @smoke @critical', ...);
test('Profile update @regression', ...);
test('Dark mode toggle @accessibility', ...);

// Run by tag
npx playwright test --grep @smoke
npx playwright test --grep @regression
```

| Tag | When to Use |
|-----|------------|
| `@smoke` | Critical path tests for release validation |
| `@regression` | Full regression suite |
| `@critical` | Business-critical flows |
| `@flaky` | Known intermittent failures under investigation |
| `@accessibility` | Accessibility-specific tests |
| `@api` | API-only tests |
| `@wip` | Work in progress (remove before merge) |

---

## 🟢 Best Practices (Strongly Recommended)

### Use `test.step()` for Grouping

```typescript
test('complete checkout flow', async ({ page }) => {
  await test.step('Add item to cart', async () => {
    await productPage.addToCart();
  });

  await test.step('Proceed to checkout', async () => {
    await cartPage.checkout();
  });

  await test.step('Complete payment', async () => {
    await checkoutPage.enterPaymentDetails(cardData);
    await checkoutPage.confirmOrder();
  });

  await test.step('Verify order confirmation', async () => {
    await expect(confirmationPage.successMessage).toBeVisible();
  });
});
```

### Use `test.describe.parallel()` Wisely

```typescript
// Safe for parallel: tests using different data/users
test.describe.parallel('Independent Product Tests', () => {
  test('product search works', ...);
  test('product filter works', ...);
  test('product detail page loads', ...);
});
```

### Retry Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,  // Retry on CI only
});
```

### Screenshot & Video on Failure

```typescript
use: {
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
  trace: 'on-first-retry',
}
```

---

## Code Review Checklist

Before merging any PR, verify:

- [ ] No `waitForTimeout` used anywhere
- [ ] All UI interactions go through page objects
- [ ] No hardcoded credentials or sensitive data
- [ ] Tests are independent (can run in any order)
- [ ] No `test.only()` or `test.skip()` left in code
- [ ] Selectors follow priority order
- [ ] Test IDs follow naming convention `[TC-XXX]`
- [ ] New env vars added to `.env.example`
- [ ] Appropriate tags applied (`@smoke`, `@regression`, etc.)
- [ ] `test.step()` used for multi-step flows
