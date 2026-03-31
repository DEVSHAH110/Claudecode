# CLAUDE.md — Playwright Test Automation Framework

## Project Overview

This is a general-purpose Playwright test automation framework designed for end-to-end (E2E), UI, and API testing. Claude (AI assistant) should use this file to understand the project structure, conventions, and how to assist effectively.

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| [Playwright](https://playwright.dev/) | Core browser automation and test runner |
| TypeScript | Primary language for all test code |
| Node.js (v18+) | Runtime environment |
| dotenv | Environment variable management |
| Allure / HTML Reporter | Test reporting |
| ESLint + Prettier | Code quality and formatting |

---

## Project Structure

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
├── .env                        # Local environment variables (gitignored)
├── playwright.config.ts        # Main Playwright configuration
├── CLAUDE.md                   # This file
├── qaagents.md                 # QA agent personas and responsibilities
├── hooks.md                    # Hook documentation
├── rules.md                    # Framework rules and standards
├── skills.md                   # Testing skills and techniques reference
└── commands.md                 # CLI commands reference
```

---

## Core Principles

1. **Page Object Model (POM)**: All UI interactions must go through page objects. No raw locators in test files.
2. **Atomic Tests**: Each test must be independent, self-contained, and not rely on other tests.
3. **No Hard Waits**: Never use `page.waitForTimeout()`. Use proper Playwright auto-waiting or explicit condition waits.
4. **Environment-Agnostic**: Tests must run across `dev`, `staging`, and `production` environments via config.
5. **Fail Fast & Loudly**: Tests should provide clear, actionable error messages on failure.
6. **Data-Driven**: Use external test data files or factory functions; never hardcode sensitive data.

---

## How Claude Should Help

- **Generate tests** following POM, fixture, and hook patterns defined in this framework.
- **Debug failures** by analyzing error messages, stack traces, and selector issues.
- **Refactor tests** to improve readability, reusability, and maintainability.
- **Write page objects** with clear method names and locator strategies.
- **Suggest selectors** prioritizing `data-testid` > ARIA roles > text > CSS.
- **Review code** for anti-patterns (flaky tests, hard waits, test coupling).
- Always reference `rules.md` for coding standards before generating code.
- Always reference `skills.md` for advanced testing patterns.
- Always reference `commands.md` for correct CLI usage.

---

## Environment Setup

```bash
npm install
npx playwright install --with-deps
cp .env.example .env   # Fill in your environment values
```

---

## Quick Start

```bash
# Run all tests
npm test

# Run in headed mode
npm run test:headed

# Run specific suite
npm run test:smoke

# Generate report
npm run report
```

---

## Key Contacts / Ownership

| Role | Responsibility |
|------|---------------|
| QA Lead | Framework architecture, PR reviews |
| QA Engineers | Test authoring, maintenance |
| DevOps | CI/CD pipeline integration |
| Claude AI | Pair programming, test generation, debugging |

---

## Notes for Claude

- Always use `async/await` — Playwright is fully promise-based.
- Prefer `expect(locator).toBeVisible()` over `isVisible()` for assertions.
- Use `test.step()` to group logical actions for better reporting.
- When generating fixtures, follow the pattern in `fixtures/` directory.
- Check `hooks.md` before adding any `beforeAll`, `beforeEach`, `afterAll`, or `afterEach` logic.
