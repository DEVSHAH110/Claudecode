# skills.md — Testing Skills & Techniques Reference

## Overview

This document catalogs advanced Playwright testing skills, patterns, and techniques used in this framework. Use it as a reference when implementing complex test scenarios.

---

## 1. Authentication Strategies

### Strategy A: Storage State (Fastest) ✅ Recommended

Save auth once in `globalSetup`, reuse across all tests.

```typescript
// hooks/globalSetup.ts
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto('/login');
await page.fill('[data-testid="email"]', process.env.TEST_EMAIL!);
await page.fill('[data-testid="password"]', process.env.TEST_PASSWORD!);
await page.click('[data-testid="submit"]');
await page.waitForURL('**/dashboard');
await page.context().storageState({ path: 'auth/user.json' });
await browser.close();
```

```typescript
// playwright.config.ts
use: { storageState: 'auth/user.json' }
```

### Strategy B: API Login (Fast, no UI dependency)

```typescript
// fixtures/auth.fixture.ts
export const test = base.extend({
  apiToken: async ({ request }, use) => {
    const res = await request.post('/api/auth/login', {
      data: { email: process.env.TEST_EMAIL, password: process.env.TEST_PASSWORD }
    });
    const { token } = await res.json();
    await use(token);
  }
});
```

### Strategy C: Multiple User Roles

```typescript
// auth/setup.ts — run separately per role
const roles = ['admin', 'viewer', 'editor'];

for (const role of roles) {
  await page.goto('/login');
  await page.fill('[data-testid="email"]', process.env[`${role.toUpperCase()}_EMAIL`]!);
  await page.fill('[data-testid="password"]', process.env[`${role.toUpperCase()}_PASSWORD`]!);
  await page.click('[data-testid="submit"]');
  await page.context().storageState({ path: `auth/${role}.json` });
}
```

```typescript
// In test file — switch auth state per test
test.use({ storageState: 'auth/admin.json' });
test('admin can delete users', async ({ page }) => { ... });
```

---

## 2. Network Interception & Mocking

### Intercept and Mock API Responses

```typescript
test('shows error when API returns 500', async ({ page }) => {
  // Mock the API before navigation
  await page.route('**/api/users', route => {
    route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Internal Server Error' })
    });
  });

  await page.goto('/users');
  await expect(page.getByTestId('error-banner')).toBeVisible();
  await expect(page.getByTestId('error-banner')).toHaveText('Something went wrong');
});
```

### Intercept and Modify Responses

```typescript
test('shows premium features for pro user', async ({ page }) => {
  await page.route('**/api/user/profile', async route => {
    const response = await route.fetch();
    const body = await response.json();
    // Modify response to inject premium status
    body.plan = 'premium';
    await route.fulfill({ response, body: JSON.stringify(body) });
  });

  await page.goto('/dashboard');
  await expect(page.getByTestId('premium-badge')).toBeVisible();
});
```

### Wait for Specific API Calls

```typescript
test('saves user data on submit', async ({ page }) => {
  const saveRequest = page.waitForRequest(req =>
    req.url().includes('/api/users') && req.method() === 'POST'
  );
  const saveResponse = page.waitForResponse('**/api/users');

  await page.click('[data-testid="save-btn"]');

  const request = await saveRequest;
  const response = await saveResponse;

  expect(JSON.parse(request.postData()!)).toMatchObject({ name: 'John' });
  expect(response.status()).toBe(201);
});
```

---

## 3. Visual Testing

### Full Page Screenshot Comparison

```typescript
test('homepage matches visual baseline @visual', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixelRatio: 0.02, // Allow 2% pixel difference
    fullPage: true,
  });
});
```

### Component Screenshot Comparison

```typescript
test('button component matches baseline', async ({ page }) => {
  const button = page.getByTestId('primary-button');
  await expect(button).toHaveScreenshot('primary-button.png');
});
```

### Update Baselines

```bash
npx playwright test --update-snapshots
```

---

## 4. Accessibility Testing

### Built-in ARIA Checks

```typescript
test('login form is accessible', async ({ page }) => {
  await page.goto('/login');
  
  // Form elements should have accessible labels
  await expect(page.getByLabel('Email address')).toBeVisible();
  await expect(page.getByLabel('Password')).toBeVisible();
  
  // Buttons should have accessible names
  await expect(page.getByRole('button', { name: 'Sign in' })).toBeVisible();
});
```

### Axe-Core Integration

```typescript
import AxeBuilder from '@axe-core/playwright';

test('page has no accessibility violations @accessibility', async ({ page }) => {
  await page.goto('/');
  
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a'])
    .analyze();
  
  expect(results.violations).toEqual([]);
});
```

### Keyboard Navigation

```typescript
test('modal can be closed with Escape key', async ({ page }) => {
  await page.click('[data-testid="open-modal-btn"]');
  await expect(page.getByRole('dialog')).toBeVisible();
  
  await page.keyboard.press('Escape');
  await expect(page.getByRole('dialog')).not.toBeVisible();
});

test('form can be submitted with Enter key', async ({ page }) => {
  await page.fill('[data-testid="search-input"]', 'playwright');
  await page.keyboard.press('Enter');
  await expect(page).toHaveURL(/.*search\?q=playwright/);
});
```

---

## 5. Data-Driven Testing

### Parameterized Tests

```typescript
const invalidEmails = [
  { input: 'notanemail', description: 'missing @ symbol' },
  { input: '@nodomain', description: 'missing local part' },
  { input: 'no@tld', description: 'missing TLD' },
  { input: '', description: 'empty string' },
];

for (const { input, description } of invalidEmails) {
  test(`shows validation error for ${description}`, async ({ page }) => {
    await loginPage.fillEmail(input);
    await loginPage.submitButton.click();
    await expect(loginPage.emailError).toBeVisible();
  });
}
```

### Factory Pattern for Test Data

```typescript
// data/factories/userFactory.ts
import { faker } from '@faker-js/faker';

export function createUser(overrides = {}) {
  return {
    firstName: faker.person.firstName(),
    lastName: faker.person.lastName(),
    email: faker.internet.email(),
    password: faker.internet.password({ length: 12, memorable: false }),
    role: 'viewer',
    ...overrides
  };
}

// In test:
const adminUser = createUser({ role: 'admin' });
const testUser = createUser({ email: 'specific@test.com' });
```

---

## 6. Multi-Tab & Multi-Window Testing

```typescript
test('clicking external link opens in new tab', async ({ page, context }) => {
  // Listen for new page before clicking
  const newTabPromise = context.waitForEvent('page');
  await page.click('[data-testid="external-link"]');
  const newTab = await newTabPromise;
  
  await newTab.waitForLoadState();
  expect(newTab.url()).toContain('https://external-site.com');
});
```

---

## 7. File Upload & Download

### File Upload

```typescript
test('user can upload profile picture', async ({ page }) => {
  // Set file input directly (works for hidden inputs too)
  await page.setInputFiles('[data-testid="avatar-input"]', 'data/files/avatar.jpg');
  await expect(page.getByTestId('upload-success')).toBeVisible();
});

// Multiple files
await page.setInputFiles('[data-testid="docs-input"]', [
  'data/files/doc1.pdf',
  'data/files/doc2.pdf'
]);
```

### File Download

```typescript
test('user can download report', async ({ page }) => {
  const downloadPromise = page.waitForEvent('download');
  await page.click('[data-testid="download-report-btn"]');
  const download = await downloadPromise;

  // Verify filename
  expect(download.suggestedFilename()).toMatch(/report-\d{4}-\d{2}-\d{2}\.csv/);

  // Save and verify content
  const path = await download.path();
  expect(path).toBeTruthy();
});
```

---

## 8. iFrame Handling

```typescript
test('interact with content inside iframe', async ({ page }) => {
  const iframeLocator = page.frameLocator('[data-testid="embedded-form"]');
  
  await iframeLocator.getByLabel('First Name').fill('John');
  await iframeLocator.getByRole('button', { name: 'Submit' }).click();
  await expect(iframeLocator.getByText('Thank you')).toBeVisible();
});
```

---

## 9. Mobile & Responsive Testing

```typescript
import { devices } from '@playwright/test';

test.use({ ...devices['iPhone 14'] });

test('mobile menu opens on hamburger click', async ({ page }) => {
  await page.goto('/');
  
  // Hamburger should be visible on mobile
  await expect(page.getByTestId('hamburger-menu')).toBeVisible();
  await page.click('[data-testid="hamburger-menu"]');
  await expect(page.getByTestId('mobile-nav')).toBeVisible();
});
```

---

## 10. Performance Testing

### Measure Page Load Time

```typescript
test('homepage loads within 3 seconds', async ({ page }) => {
  const startTime = Date.now();
  await page.goto('/', { waitUntil: 'load' });
  const loadTime = Date.now() - startTime;
  
  expect(loadTime).toBeLessThan(3000);
  console.log(`Page loaded in ${loadTime}ms`);
});
```

### Core Web Vitals via CDP

```typescript
test('measure LCP', async ({ page }) => {
  const client = await page.context().newCDPSession(page);
  await client.send('Performance.enable');

  await page.goto('/');

  const metrics = await client.send('Performance.getMetrics');
  const lcp = metrics.metrics.find(m => m.name === 'LargestContentfulPaint');
  
  console.log(`LCP: ${lcp?.value}ms`);
  expect(lcp?.value).toBeLessThan(2500); // Good LCP < 2.5s
});
```

---

## 11. Custom Reporter

```typescript
// reporters/customReporter.ts
import { Reporter, TestCase, TestResult } from '@playwright/test/reporter';

class CustomReporter implements Reporter {
  onTestEnd(test: TestCase, result: TestResult) {
    const status = result.status === 'passed' ? '✅' : '❌';
    console.log(`${status} ${test.title} (${result.duration}ms)`);
  }
}

export default CustomReporter;
```

---

## 12. Soft Assertions

Use when you want to collect multiple failures in one test run.

```typescript
test('dashboard widgets all load', async ({ page }) => {
  await page.goto('/dashboard');
  
  // Soft assertions — don't stop on first failure
  await expect.soft(page.getByTestId('revenue-widget')).toBeVisible();
  await expect.soft(page.getByTestId('users-widget')).toBeVisible();
  await expect.soft(page.getByTestId('orders-widget')).toBeVisible();
  await expect.soft(page.getByTestId('analytics-widget')).toBeVisible();
  
  // Hard assertion at end — fails if any soft assertion failed
  expect(test.info().errors).toHaveLength(0);
});
```

---

## 13. Test Retry & Flakiness Management

```typescript
// Annotate a test as potentially flaky
test('third-party payment widget loads @flaky', async ({ page }) => {
  test.info().annotations.push({
    type: 'flaky',
    description: 'Third-party widget has intermittent load times'
  });
  
  // ...
});

// Retry a specific test differently
test.describe('Flaky Suite', () => {
  test.describe.configure({ retries: 3 });
  
  test('sometimes fails', async ({ page }) => {
    // ...
  });
});
```
