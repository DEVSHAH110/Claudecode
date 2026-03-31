# hooks.md — Playwright Test Hooks Reference

## Overview

Hooks in Playwright allow you to run setup and teardown logic at different stages of the test lifecycle. This document defines when and how to use each hook in this framework, along with code patterns and best practices.

---

## Hook Lifecycle Order

```
test.beforeAll()          ← Runs once before all tests in a describe block
  └─ test.beforeEach()    ← Runs before each individual test
       └─ test()          ← The actual test
  └─ test.afterEach()     ← Runs after each individual test
test.afterAll()           ← Runs once after all tests in a describe block
```

---

## Global Hooks (via `globalSetup` / `globalTeardown`)

Defined in `playwright.config.ts`. These run once for the **entire test suite**.

### globalSetup

**File**: `hooks/globalSetup.ts`  
**When to use**: One-time setup before any test runs.

**Use cases**:
- Authenticate and save storage state
- Seed test database
- Start mock servers
- Set global environment variables

```typescript
// hooks/globalSetup.ts
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  // Authenticate once and save state
  await page.goto(process.env.BASE_URL + '/login');
  await page.fill('[data-testid="email"]', process.env.TEST_USER_EMAIL!);
  await page.fill('[data-testid="password"]', process.env.TEST_USER_PASSWORD!);
  await page.click('[data-testid="login-btn"]');
  await page.waitForURL('**/dashboard');
  await page.context().storageState({ path: 'auth/user.json' });

  await browser.close();
}

export default globalSetup;
```

---

### globalTeardown

**File**: `hooks/globalTeardown.ts`  
**When to use**: One-time cleanup after all tests complete.

**Use cases**:
- Clean up test data from database
- Stop mock servers
- Archive logs

```typescript
// hooks/globalTeardown.ts
async function globalTeardown() {
  // Example: clean up seeded test data
  await cleanupTestDatabase();
  console.log('✅ Global teardown complete');
}

export default globalTeardown;
```

---

## Test-Level Hooks

Used inside `test.describe()` blocks in test files.

---

### `test.beforeAll()`

**Scope**: Runs once before all tests in the enclosing `describe` block.  
**Worker**: Runs in the same worker as the tests (not shared across workers).

**When to use**:
- Set up shared resources (e.g., create a test user, start a stub)
- Initialize data that all tests in the block will share (read-only)

**When NOT to use**:
- Do NOT use if tests mutate the shared state — use `beforeEach` instead
- Do NOT use for browser/page setup — Playwright handles this

```typescript
import { test, expect } from '@playwright/test';

test.describe('User Management', () => {
  let userId: string;

  test.beforeAll(async ({ request }) => {
    // Create a shared test user via API
    const response = await request.post('/api/users', {
      data: { email: 'test@example.com', role: 'admin' }
    });
    const user = await response.json();
    userId = user.id;
  });

  test.afterAll(async ({ request }) => {
    // Clean up the shared test user
    await request.delete(`/api/users/${userId}`);
  });

  test('admin can view users list', async ({ page }) => {
    // userId is available here
  });
});
```

---

### `test.beforeEach()`

**Scope**: Runs before every individual test in the `describe` block.  
**When to use**: This is the most commonly used hook.

**Use cases**:
- Navigate to the starting page
- Log in a user (if not using stored auth state)
- Set up fresh test data per test
- Reset application state

```typescript
test.describe('Shopping Cart', () => {
  test.beforeEach(async ({ page }) => {
    // Navigate to the starting point before each test
    await page.goto('/shop');
    await page.waitForLoadState('networkidle');
  });

  test('user can add item to cart', async ({ page }) => {
    // Page is already at /shop
    await page.click('[data-testid="add-to-cart-btn"]');
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');
  });

  test('user can remove item from cart', async ({ page }) => {
    // Fresh start — page is at /shop again
  });
});
```

---

### `test.afterEach()`

**Scope**: Runs after every individual test, regardless of pass/fail.  
**When to use**: Per-test cleanup and diagnostics.

**Use cases**:
- Take screenshots on failure
- Clean up test-specific data
- Log test results
- Reset state changed during the test

```typescript
test.afterEach(async ({ page }, testInfo) => {
  if (testInfo.status !== testInfo.expectedStatus) {
    // Capture screenshot on failure
    const screenshot = await page.screenshot();
    await testInfo.attach('screenshot-on-failure', {
      body: screenshot,
      contentType: 'image/png'
    });
  }
});
```

---

### `test.afterAll()`

**Scope**: Runs once after all tests in the `describe` block.  
**When to use**: Clean up shared resources created in `beforeAll`.

**Use cases**:
- Delete test users/data created in `beforeAll`
- Close connections
- Generate summary reports

```typescript
test.afterAll(async ({ request }) => {
  await request.delete('/api/test-data/cleanup');
});
```

---

## Fixture-Based Hooks (Recommended Pattern)

Instead of hooks, prefer **Playwright fixtures** for setup/teardown. Fixtures are more composable, reusable, and easier to debug.

**File**: `fixtures/index.ts`

```typescript
// fixtures/index.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';

type MyFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  authenticatedPage: void;
};

export const test = base.extend<MyFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },

  dashboardPage: async ({ page }, use) => {
    const dashboardPage = new DashboardPage(page);
    await use(dashboardPage);
  },

  // Fixture with built-in setup + teardown
  authenticatedPage: async ({ page }, use) => {
    // SETUP: login before test
    await page.goto('/login');
    await page.fill('[data-testid="email"]', process.env.TEST_EMAIL!);
    await page.fill('[data-testid="password"]', process.env.TEST_PASSWORD!);
    await page.click('[data-testid="submit"]');
    await page.waitForURL('**/dashboard');

    await use(); // test runs here

    // TEARDOWN: logout after test
    await page.click('[data-testid="logout"]');
  }
});

export { expect };
```

**Usage in tests**:
```typescript
import { test, expect } from '../fixtures';

test('dashboard loads after login', async ({ authenticatedPage, dashboardPage }) => {
  await expect(dashboardPage.heading).toBeVisible();
});
```

---

## Hook Anti-Patterns ❌

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| `page.waitForTimeout(3000)` in hooks | Causes slow, flaky tests | Use `waitForLoadState()` or `waitForSelector()` |
| Shared mutable state across tests in `beforeAll` | Tests become order-dependent | Use `beforeEach` or isolate data per test |
| Login inside every `beforeEach` via UI | Slow — adds seconds per test | Use `storageState` for auth |
| Asserting in `afterEach` | Hides the real failure | Put assertions in the test itself |
| Not cleaning up in `afterAll/afterEach` | Pollutes test environment | Always pair setup with teardown |

---

## Hook Execution in Parallel Workers

Playwright runs test files in parallel across workers by default.

```
Worker 1: beforeAll → [test1, test2] → afterAll
Worker 2: beforeAll → [test3, test4] → afterAll
```

- `globalSetup/globalTeardown` run **once** across all workers.
- `beforeAll/afterAll` run **once per worker** per describe block.
- `beforeEach/afterEach` run **per test** in each worker.

> ⚠️ Never share mutable data between workers without locking mechanisms.

---

## Configuration Reference

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  globalSetup: './hooks/globalSetup.ts',
  globalTeardown: './hooks/globalTeardown.ts',
  use: {
    storageState: 'auth/user.json', // Apply saved auth to all tests
    baseURL: process.env.BASE_URL,
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
  },
});
```
