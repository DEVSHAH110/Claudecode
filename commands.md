# commands.md — CLI Commands Reference

## Overview

Complete reference for all CLI commands used in this Playwright test automation framework. Commands are organized by category.

---

## Installation & Setup

```bash
# Install all dependencies
npm install

# Install Playwright browsers + system dependencies
npx playwright install --with-deps

# Install only specific browsers
npx playwright install chromium
npx playwright install firefox
npx playwright install webkit

# Install system dependencies only (CI environments)
npx playwright install-deps

# Copy environment config
cp .env.example .env
```

---

## Running Tests

### Run All Tests

```bash
# Run all tests (headless, default config)
npx playwright test

# Using npm script alias
npm test
```

### Run by File or Folder

```bash
# Run a specific test file
npx playwright test tests/e2e/login.spec.ts

# Run all tests in a folder
npx playwright test tests/e2e/
npx playwright test tests/api/
npx playwright test tests/smoke/

# Run multiple specific files
npx playwright test tests/e2e/login.spec.ts tests/e2e/checkout.spec.ts
```

### Run by Test Name or Tag

```bash
# Run tests matching a string (partial match)
npx playwright test --grep "login"
npx playwright test --grep "should redirect to dashboard"

# Run by tag
npx playwright test --grep "@smoke"
npx playwright test --grep "@regression"
npx playwright test --grep "@critical"
npx playwright test --grep "@accessibility"

# Exclude tests matching a tag
npx playwright test --grep-invert "@flaky"

# Combine tags (AND logic)
npx playwright test --grep "(?=.*@smoke)(?=.*@critical)"
```

### Run by Browser

```bash
# Run on Chromium only
npx playwright test --project=chromium

# Run on Firefox only
npx playwright test --project=firefox

# Run on WebKit (Safari) only
npx playwright test --project=webkit

# Run on multiple browsers
npx playwright test --project=chromium --project=firefox

# Run on mobile viewports
npx playwright test --project="Mobile Chrome"
npx playwright test --project="Mobile Safari"
```

### Run in Headed Mode (Visible Browser)

```bash
# Run with browser window visible
npx playwright test --headed

# Run headed on specific browser
npx playwright test --headed --project=chromium
```

### Run with Parallelism Control

```bash
# Run tests sequentially (no parallel workers)
npx playwright test --workers=1

# Run with specific number of workers
npx playwright test --workers=4

# Run with max workers (default behavior)
npx playwright test --workers=100%
```

### Run with Retries

```bash
# Retry failed tests up to 2 times
npx playwright test --retries=2

# No retries (override config)
npx playwright test --retries=0
```

### Run with Timeout Override

```bash
# Set test timeout to 60 seconds
npx playwright test --timeout=60000

# Disable timeout
npx playwright test --timeout=0
```

---

## Debug Mode

### Interactive Debug (Step Through Tests)

```bash
# Open Playwright Inspector for debugging
npx playwright test --debug

# Debug a specific test file
npx playwright test tests/e2e/login.spec.ts --debug

# Debug a specific test by name
npx playwright test --debug --grep "should login successfully"
```

### Pause Inside a Test

Add this line inside your test to pause execution:
```typescript
await page.pause(); // Opens Inspector at this point
```

### UI Mode (Interactive Test Runner) ✅ Recommended for Development

```bash
# Open Playwright UI Mode
npx playwright test --ui

# UI mode with specific port
npx playwright test --ui --ui-port=8080
```

### Slow Motion

```bash
# Run tests in slow motion (500ms between actions)
npx playwright test --headed --slowmo=500

# Very slow (good for recording demos)
npx playwright test --headed --slowmo=1500
```

### Verbose Output

```bash
# Show detailed logs
npx playwright test --reporter=line

# Show all console output
DEBUG=pw:api npx playwright test
```

---

## Trace & Artifacts

### Record Traces

```bash
# Run with trace recording
npx playwright test --trace=on

# Record trace on first retry only
npx playwright test --trace=on-first-retry

# Record trace only on failure
npx playwright test --trace=retain-on-failure
```

### View Traces

```bash
# Open the Playwright Trace Viewer
npx playwright show-trace test-results/trace.zip

# Open trace from a specific test result
npx playwright show-trace test-results/login-spec-ts-should-login/trace.zip
```

### Screenshots & Videos

```bash
# Capture screenshots on failure (override config)
npx playwright test --screenshot=only-on-failure

# Always capture screenshots
npx playwright test --screenshot=on

# Record video
npx playwright test --video=on
npx playwright test --video=retain-on-failure
```

---

## Reporting

### Generate Reports

```bash
# Run tests and open HTML report automatically
npx playwright test --reporter=html

# Generate report without auto-opening
npx playwright test --reporter=html,line

# Generate JUnit XML (for CI)
npx playwright test --reporter=junit --output=results/junit.xml

# Generate JSON report
npx playwright test --reporter=json --output=results/report.json

# Multiple reporters at once
npx playwright test --reporter=html,junit,line
```

### Open HTML Report

```bash
# Open last generated HTML report
npx playwright show-report

# Open report from specific folder
npx playwright show-report playwright-report/

# Open on specific port
npx playwright show-report --host=0.0.0.0 --port=9323
```

### Allure Report (if configured)

```bash
# Generate Allure report
npx allure generate allure-results --clean -o allure-report

# Open Allure report
npx allure open allure-report
```

---

## Code Generation (Codegen)

```bash
# Record a new test by interacting with the browser
npx playwright codegen https://example.com

# Record with a specific viewport size
npx playwright codegen --viewport-size=1280,720 https://example.com

# Record and save to a file
npx playwright codegen --output=tests/e2e/recorded.spec.ts https://example.com

# Record with specific browser
npx playwright codegen --browser=firefox https://example.com

# Record with authentication state
npx playwright codegen --load-storage=auth/user.json https://example.com

# Record with custom device (mobile)
npx playwright codegen --device="iPhone 14" https://example.com
```

---

## Visual Testing (Snapshots)

```bash
# Update visual baselines (run after intentional UI changes)
npx playwright test --update-snapshots

# Update snapshots for specific test
npx playwright test tests/visual/homepage.spec.ts --update-snapshots
```

---

## Environment-Specific Runs

```bash
# Run against staging environment
BASE_URL=https://staging.example.com npx playwright test

# Run against production (smoke only)
BASE_URL=https://example.com npx playwright test --grep "@smoke"

# Using dotenv files per environment
npx dotenv -e .env.staging -- npx playwright test
npx dotenv -e .env.production -- npx playwright test --grep "@smoke"
```

---

## CI/CD Commands

```bash
# Full CI run (no headed, with retries, JUnit output)
npx playwright test \
  --workers=4 \
  --retries=2 \
  --reporter=junit,html \

# Smoke test gate for deployment
npx playwright test \
  --grep "@smoke" \
  --workers=2 \
  --retries=1 \
  --reporter=line

# Shard tests across multiple CI machines
# Machine 1 of 3:
npx playwright test --shard=1/3

# Machine 2 of 3:
npx playwright test --shard=2/3

# Machine 3 of 3:
npx playwright test --shard=3/3
```

---

## Maintenance Commands

```bash
# Check Playwright version
npx playwright --version

# List all available projects (browsers/devices)
npx playwright test --list --reporter=line

# List all test names without running
npx playwright test --list

# Clear test results and cache
rm -rf test-results/ playwright-report/ .playwright-cache/

# Lint and format test code
npm run lint
npm run format

# Type-check TypeScript
npm run type-check

# Check for outdated dependencies
npm outdated

# Update Playwright to latest
npm install @playwright/test@latest
npx playwright install
```

---

## NPM Script Aliases

Defined in `package.json` for convenience:

```json
{
  "scripts": {
    "test": "npx playwright test",
    "test:headed": "npx playwright test --headed",
    "test:debug": "npx playwright test --debug",
    "test:ui": "npx playwright test --ui",
    "test:smoke": "npx playwright test --grep @smoke",
    "test:regression": "npx playwright test --grep @regression",
    "test:api": "npx playwright test tests/api/",
    "test:chromium": "npx playwright test --project=chromium",
    "test:firefox": "npx playwright test --project=firefox",
    "test:webkit": "npx playwright test --project=webkit",
    "test:mobile": "npx playwright test --project='Mobile Chrome'",
    "test:update-snapshots": "npx playwright test --update-snapshots",
    "report": "npx playwright show-report",
    "codegen": "npx playwright codegen",
    "trace": "npx playwright show-trace",
    "lint": "eslint . --ext .ts",
    "format": "prettier --write .",
    "type-check": "tsc --noEmit"
  }
}
```

---

## Quick Reference Card

| Goal | Command |
|------|---------|
| Run all tests | `npm test` |
| Run smoke tests | `npm run test:smoke` |
| Debug a test | `npm run test:debug` |
| Interactive UI | `npm run test:ui` |
| Open report | `npm run report` |
| Record a test | `npm run codegen` |
| Update snapshots | `npm run test:update-snapshots` |
| Run on Firefox | `npm run test:firefox` |
| Run API tests | `npm run test:api` |
