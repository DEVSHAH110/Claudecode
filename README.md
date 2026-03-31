# 🎭 Playwright Test Automation Framework

A general-purpose, production-ready Playwright framework for **End-to-End (E2E)**, **UI**, and **API** testing — built with TypeScript, structured around the Page Object Model, and designed to scale across teams.

---

## 📁 Repository Structure

```
playwright-framework/
├── .github/                    # CI/CD workflows (GitHub Actions)
├── tests/
│   ├── e2e/                    # End-to-end UI tests
│   ├── api/                    # API tests
│   ├── components/             # Component-level tests
│   └── smoke/                  # Smoke test suite
├── pages/                      # Page Object Models (POM)
├── fixtures/                   # Custom Playwright fixtures
├── helpers/                    # Utility/helper functions
├── data/                       # Test data (JSON, CSV, factories)
├── config/                     # Environment configs
├── hooks/                      # Global before/after hooks
├── auth/                       # Saved authentication states
├── .env.example                # Environment variable template
├── playwright.config.ts        # Main Playwright configuration
├── CLAUDE.md                   # AI assistant context file
├── qaagents.md                 # QA agent personas & responsibilities
├── hooks.md                    # Hook lifecycle documentation
├── rules.md                    # Framework rules & coding standards
├── skills.md                   # Advanced testing techniques
├── commands.md                 # CLI commands reference
└── README.md                   # This file
```

---

## 🧰 Tech Stack

| Tool | Purpose |
|------|---------|
| [Playwright](https://playwright.dev/) | Core browser automation and test runner |
| TypeScript | Primary language for all test code |
| Node.js (v18+) | Runtime environment |
| dotenv | Environment variable management |
| Allure / HTML Reporter | Test reporting |
| ESLint + Prettier | Code quality and formatting |

---

## ⚡ Quick Start

### 1. Install dependencies

```bash
npm install
npx playwright install --with-deps
cp .env.example .env   # Fill in your environment values
```

### 2. Run tests

```bash
npm test                    # Run all tests
npm run test:headed         # Run with visible browser
npm run test:smoke          # Run smoke suite only
npm run test:debug          # Debug mode with Inspector
npm run test:ui             # Interactive UI mode
npm run report              # Open HTML report
```

---

## 🔑 Core Principles

1. **Page Object Model (POM)** — All UI interactions go through page objects. No raw locators in test files.
2. **Atomic Tests** — Each test is independent, self-contained, and does not rely on other tests.
3. **No Hard Waits** — Never use `page.waitForTimeout()`. Use Playwright's built-in auto-waiting.
4. **Environment-Agnostic** — Tests run across `dev`, `staging`, and `production` via config.
5. **Fail Fast & Loudly** — Tests provide clear, actionable error messages on failure.
6. **Data-Driven** — External test data files and factory functions; never hardcode sensitive data.

---

## 📄 Documentation Files

---

### 📘 CLAUDE.md — AI Assistant Context

Provides Claude AI (and any contributor) with the full project context: tech stack, folder structure, core principles, and how AI should assist in this framework.

**How Claude uses this file:**
- Generates tests following POM, fixture, and hook patterns
- Debugs failures by analyzing errors, stack traces, and selectors
- Refactors tests for readability and maintainability
- Writes page objects with correct locator strategies
- Reviews code for anti-patterns (flaky tests, hard waits, coupling)

**Key rules Claude always follows:**
- Use `async/await` — Playwright is fully promise-based
- Prefer `expect(locator).toBeVisible()` over `isVisible()` for assertions
- Use `test.step()` to group logical actions for better reporting
- Always check `rules.md` before generating any code
- Always check `hooks.md` before adding setup/teardown logic

---

### 👥 qaagents.md — QA Agent Personas & Responsibilities

Defines the six QA agent roles within the framework — used to organize team responsibilities and guide AI-assisted workflows.

| Agent | Role | Primary Ownership |
|-------|------|------------------|
| 🔵 Framework Architect | Infrastructure, config, standards | `playwright.config.ts`, `fixtures/`, `hooks/` |
| 🟢 E2E Test Engineer | User journey and UI test authoring | `tests/e2e/`, `pages/`, `data/` |
| 🟡 API Test Engineer | Backend contract and integration tests | `tests/api/`, `helpers/apiClient.ts` |
| 🟠 Smoke & Regression Gatekeeper | Release quality gates and tagging | `tests/smoke/`, CI config |
| 🔴 Data & Environment Manager | Test data, env configs, isolation | `data/`, `config/`, `.env.example` |
| 🤖 Claude AI | Pair programming, code generation, review | All files (assist only) |

**Agent Interaction Flow:**
```
Product/Dev → Requirements → E2E Agent + API Agent
                                    ↓
                              Claude AI (assists)
                                    ↓
                           Framework Architect (reviews)
                                    ↓
                         Smoke Gatekeeper (release gate)
                                    ↓
                          Data/Env Manager (supports)
```

**Escalation Path:**

| Issue | First Contact | Escalate To |
|-------|-------------|-------------|
| Flaky test | E2E / API Agent | Framework Architect |
| Environment down | Data/Env Manager | DevOps |
| Release blocker | Smoke Gatekeeper | QA Lead |
| Framework bug | Framework Architect | QA Lead |

---

### 🪝 hooks.md — Test Hook Lifecycle

Documents all Playwright hooks and when to use them across the test lifecycle.

#### Hook Execution Order

```
globalSetup()              ← Once before the entire test suite
  test.beforeAll()         ← Once before all tests in a describe block
    test.beforeEach()      ← Before each individual test
      test()               ← The actual test
    test.afterEach()       ← After each individual test
  test.afterAll()          ← Once after all tests in a describe block
globalTeardown()           ← Once after the entire test suite
```

#### Hook Summary

| Hook | Scope | Common Use Cases |
|------|-------|-----------------|
| `globalSetup` | Entire suite (once) | Auth state, DB seed, mock server start |
| `globalTeardown` | Entire suite (once) | DB cleanup, stop servers, archive logs |
| `test.beforeAll` | Describe block (once) | Create shared test users, init shared data |
| `test.beforeEach` | Per test | Navigate to page, reset state, fresh data |
| `test.afterEach` | Per test | Screenshots on failure, cleanup per-test data |
| `test.afterAll` | Describe block (once) | Delete shared users, close connections |

#### Recommended: Fixture-Based Setup (Preferred over hooks)

```typescript
// fixtures/index.ts
export const test = base.extend<MyFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // SETUP: login before test
    await page.goto('/login');
    await page.fill('[data-testid="email"]', process.env.TEST_EMAIL!);
    await page.fill('[data-testid="password"]', process.env.TEST_PASSWORD!);
    await page.click('[data-testid="submit"]');
    await page.waitForURL('**/dashboard');

    await use(); // ← test runs here

    // TEARDOWN: logout after test
    await page.click('[data-testid="logout"]');
  }
});
```

#### Hook Anti-Patterns ❌

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `waitForTimeout()` in hooks | Flaky and slow | Use `waitForLoadState()` or `waitForSelector()` |
| Shared mutable state in `beforeAll` | Test order dependency | Use `beforeEach` or isolate per-test data |
| UI login in every `beforeEach` | Slow test runs | Use `storageState` for authentication |
| Assertions in `afterEach` | Hides real failures | Put assertions inside the test itself |
| No teardown after `beforeAll` | Environment pollution | Always pair setup with matching teardown |

---

### 📏 rules.md — Framework Rules & Coding Standards

Mandatory conventions for all contributors. Non-negotiable rules enforced at code review.

#### 🔴 Non-Negotiable Rules

**R1 — No Hard Waits**
```typescript
// ❌ NEVER
await page.waitForTimeout(3000);

// ✅ ALWAYS
await page.waitForLoadState('networkidle');
await expect(locator).toBeVisible();
```

**R2 — No Raw Locators in Test Files** — All UI interactions go through Page Object classes.

**R3 — No Hardcoded Credentials** — Always use `process.env.VARIABLE_NAME!`

**R4 — Every Test Must Be Independent** — Tests must not depend on execution order or shared mutable state.

**R5 — No `test.only()` or `test.skip()` on Main Branch** — Local development only; remove before committing.

**R6 — TypeScript Strict Mode Only** — All files must be `.ts`. No `.js` files in `tests/`, `pages/`, `fixtures/`, or `helpers/`.

#### Selector Priority Order

```
1. data-testid          →  [data-testid="login-btn"]           ← BEST
2. ARIA role + name     →  getByRole('button', { name: 'Submit' })
3. Accessible label     →  getByLabel('Email address')
4. Visible text         →  getByText('Sign in')
5. CSS class            →  .login-button                       ← LAST RESORT
```
Never use: XPath, auto-generated IDs, or positional selectors.

#### File Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Test files | `*.spec.ts` | `login.spec.ts` |
| Page Objects | `*Page.ts` (PascalCase) | `LoginPage.ts` |
| Fixtures | `*.fixture.ts` | `auth.fixture.ts` |
| Helpers | `*Helper.ts` / `*.utils.ts` | `apiHelper.ts` |
| Test data | `*.data.ts` / `*.json` | `users.data.ts` |

#### Test Naming Convention

```typescript
test('[TC-001] should redirect to dashboard when valid credentials provided', ...);
test('[TC-002] should display error when invalid password entered', ...);
```

#### Test Tagging

| Tag | Usage |
|-----|-------|
| `@smoke` | Critical path — run before every release |
| `@regression` | Full regression suite |
| `@critical` | Business-critical flows |
| `@flaky` | Known intermittent failures under investigation |
| `@accessibility` | Accessibility-specific tests |
| `@api` | API-only tests |
| `@wip` | Work in progress (remove before merge) |

#### Code Review Checklist

- [ ] No `waitForTimeout` anywhere
- [ ] All UI interactions go through page objects
- [ ] No hardcoded credentials or sensitive data
- [ ] Tests are independent (run in any order)
- [ ] No `test.only()` or `test.skip()` committed
- [ ] Selectors follow priority order
- [ ] Test IDs follow `[TC-XXX]` naming convention
- [ ] New env vars added to `.env.example`
- [ ] Appropriate tags applied
- [ ] `test.step()` used for multi-step flows

---

### 🧠 skills.md — Advanced Testing Techniques

Reference for 13 advanced Playwright patterns used in this framework.

#### 1. Authentication Strategies

| Strategy | Speed | Use Case |
|----------|-------|---------|
| Storage State (saved auth) | ⚡ Fastest | Default for all UI tests |
| API Login | 🚀 Fast | When UI auth is unreliable |
| Multiple Role Auth | 🔑 Per-role | Admin/viewer/editor testing |

```typescript
// Fastest: save auth once in globalSetup, reuse everywhere
await page.context().storageState({ path: 'auth/user.json' });
// playwright.config.ts:
use: { storageState: 'auth/user.json' }
```

#### 2. Network Interception & Mocking

```typescript
// Mock a failing API
await page.route('**/api/users', route => route.fulfill({
  status: 500,
  body: JSON.stringify({ error: 'Server Error' })
}));

// Intercept and modify real responses
await page.route('**/api/profile', async route => {
  const response = await route.fetch();
  const body = await response.json();
  body.plan = 'premium';
  await route.fulfill({ response, body: JSON.stringify(body) });
});

// Wait for specific request/response
const saveResponse = page.waitForResponse('**/api/users');
await page.click('[data-testid="save-btn"]');
expect((await saveResponse).status()).toBe(201);
```

#### 3. Visual Testing (Screenshot Comparison)

```typescript
await expect(page).toHaveScreenshot('homepage.png', {
  maxDiffPixelRatio: 0.02,
  fullPage: true,
});
// Update baselines: npx playwright test --update-snapshots
```

#### 4. Accessibility Testing

```typescript
import AxeBuilder from '@axe-core/playwright';

const results = await new AxeBuilder({ page })
  .withTags(['wcag2a', 'wcag2aa'])
  .analyze();
expect(results.violations).toEqual([]);
```

#### 5. Data-Driven Testing (Factory Pattern)

```typescript
// data/factories/userFactory.ts
import { faker } from '@faker-js/faker';

export function createUser(overrides = {}) {
  return {
    email: faker.internet.email(),
    password: faker.internet.password({ length: 12 }),
    role: 'viewer',
    ...overrides
  };
}
```

#### 6. Multi-Tab Testing

```typescript
const newTabPromise = context.waitForEvent('page');
await page.click('[data-testid="external-link"]');
const newTab = await newTabPromise;
await newTab.waitForLoadState();
expect(newTab.url()).toContain('https://external-site.com');
```

#### 7. File Upload & Download

```typescript
// Upload
await page.setInputFiles('[data-testid="avatar-input"]', 'data/files/avatar.jpg');

// Download
const downloadPromise = page.waitForEvent('download');
await page.click('[data-testid="download-btn"]');
const download = await downloadPromise;
expect(download.suggestedFilename()).toMatch(/report.*\.csv/);
```

#### 8. iFrame Handling

```typescript
const frame = page.frameLocator('[data-testid="embedded-form"]');
await frame.getByLabel('First Name').fill('John');
await frame.getByRole('button', { name: 'Submit' }).click();
```

#### 9. Mobile & Responsive Testing

```typescript
import { devices } from '@playwright/test';
test.use({ ...devices['iPhone 14'] });
```

#### 10. Soft Assertions (Collect Multiple Failures)

```typescript
await expect.soft(page.getByTestId('widget-1')).toBeVisible();
await expect.soft(page.getByTestId('widget-2')).toBeVisible();
await expect.soft(page.getByTestId('widget-3')).toBeVisible();
expect(test.info().errors).toHaveLength(0); // fail once at end
```

#### 11. `test.step()` for Grouped Reporting

```typescript
test('complete checkout flow', async ({ page }) => {
  await test.step('Add item to cart', async () => { ... });
  await test.step('Proceed to checkout', async () => { ... });
  await test.step('Complete payment', async () => { ... });
  await test.step('Verify confirmation', async () => { ... });
});
```

#### 12. Performance Measurement

```typescript
const startTime = Date.now();
await page.goto('/', { waitUntil: 'load' });
expect(Date.now() - startTime).toBeLessThan(3000);
```

#### 13. CI Test Sharding

```typescript
// Split tests across 3 CI machines
npx playwright test --shard=1/3
npx playwright test --shard=2/3
npx playwright test --shard=3/3
```

---

### ⌨️ commands.md — CLI Commands Reference

Complete reference for all framework CLI commands.

#### Installation

```bash
npm install                                  # Install dependencies
npx playwright install --with-deps          # Install browsers + system deps
npx playwright install chromium             # Install specific browser only
```

#### Running Tests

```bash
# By scope
npx playwright test                          # All tests
npx playwright test tests/e2e/login.spec.ts  # Specific file
npx playwright test tests/e2e/               # Entire folder
npx playwright test --grep "login"           # Match test name
npx playwright test --grep "@smoke"          # By tag
npx playwright test --grep-invert "@flaky"  # Exclude tag

# By browser
npx playwright test --project=chromium
npx playwright test --project=firefox
npx playwright test --project=webkit
npx playwright test --project="Mobile Chrome"

# Execution control
npx playwright test --headed                 # Visible browser
npx playwright test --workers=4             # Parallel workers
npx playwright test --retries=2             # Retry on failure
npx playwright test --timeout=60000         # Override timeout
```

#### Debug Mode

```bash
npx playwright test --debug                  # Playwright Inspector
npx playwright test --ui                     # Interactive UI Mode ✅
npx playwright test --headed --slowmo=500   # Slow motion
```

#### Trace & Artifacts

```bash
npx playwright test --trace=on               # Record all traces
npx playwright test --trace=retain-on-failure
npx playwright show-trace test-results/trace.zip
npx playwright test --screenshot=only-on-failure
npx playwright test --video=retain-on-failure
```

#### Reporting

```bash
npx playwright test --reporter=html          # HTML report
npx playwright test --reporter=junit         # JUnit XML (CI)
npx playwright show-report                   # Open last report
```

#### Code Generation

```bash
npx playwright codegen https://example.com                         # Record test
npx playwright codegen --output=tests/recorded.spec.ts [url]      # Save to file
npx playwright codegen --load-storage=auth/user.json [url]        # With auth state
npx playwright codegen --device="iPhone 14" [url]                 # Mobile recording
```

#### Visual Snapshots

```bash
npx playwright test --update-snapshots      # Update visual baselines
```

#### Environment-Specific Runs

```bash
BASE_URL=https://staging.example.com npx playwright test
BASE_URL=https://example.com npx playwright test --grep "@smoke"
```

#### CI/CD Commands

```bash
# Full CI run
npx playwright test --workers=4 --retries=2 --reporter=junit,html

# Smoke gate for deployment
npx playwright test --grep "@smoke" --workers=2 --retries=1

# Sharded run (distributed CI)
npx playwright test --shard=1/3   # Machine 1
npx playwright test --shard=2/3   # Machine 2
npx playwright test --shard=3/3   # Machine 3
```

#### NPM Script Aliases

| Script | Command |
|--------|---------|
| `npm test` | Run all tests |
| `npm run test:headed` | Run with visible browser |
| `npm run test:debug` | Debug with Inspector |
| `npm run test:ui` | Interactive UI mode |
| `npm run test:smoke` | Smoke suite only |
| `npm run test:regression` | Full regression suite |
| `npm run test:api` | API tests only |
| `npm run test:chromium` | Chromium only |
| `npm run test:firefox` | Firefox only |
| `npm run test:webkit` | WebKit / Safari |
| `npm run test:mobile` | Mobile Chrome viewport |
| `npm run test:update-snapshots` | Update visual baselines |
| `npm run report` | Open HTML report |
| `npm run codegen` | Launch test recorder |
| `npm run lint` | Lint TypeScript files |
| `npm run format` | Format with Prettier |
| `npm run type-check` | TypeScript type validation |

---

## 🌍 Environment Variables

All environment variables must be declared in `.env.example`. Copy to `.env` and fill in values before running locally.

```bash
# .env.example
BASE_URL=https://your-app-url.com
TEST_EMAIL=testuser@example.com
TEST_PASSWORD=your_test_password
API_KEY=your_api_key
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=admin_password
```

Variables are validated at startup via `helpers/envValidator.ts`. Missing required variables will throw a clear error before any test runs.

---

## 🔄 CI/CD Integration

Tests are designed to run in any CI environment. Recommended GitHub Actions setup:

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 18 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test --workers=4 --retries=2
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## 📬 Team Contacts

| Role | Responsibility |
|------|---------------|
| QA Lead | Framework architecture, PR reviews |
| QA Engineers | Test authoring, maintenance |
| DevOps | CI/CD pipeline integration |
| Claude AI | Pair programming, test generation, debugging |

---

## 📚 Resources

- [Playwright Official Docs](https://playwright.dev/docs/intro)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [Axe-core for Accessibility](https://github.com/dequelabs/axe-core)
- [Faker.js for Test Data](https://fakerjs.dev/)

---

<div align="center">

Built with ❤️ using [Playwright](https://playwright.dev/) + TypeScript

</div>
