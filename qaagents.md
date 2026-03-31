# qaagents.md — QA Agent Personas & Responsibilities

## Overview

This file defines the QA agent roles within the Playwright framework. Each agent has a specific persona, scope of responsibility, and set of tasks they own. This is used to organize human team members and to guide AI-assisted test generation and review.

---

## Agent Definitions

---

### 🔵 Agent: Framework Architect

**Persona**: Senior QA Engineer / Automation Lead  
**Goal**: Maintain and evolve the test automation framework infrastructure.

**Responsibilities**:
- Design and maintain `playwright.config.ts`
- Define global fixtures, hooks, and shared utilities
- Set coding standards (see `rules.md`)
- Manage CI/CD pipeline integration
- Onboard new engineers and review PRs
- Upgrade Playwright and dependency versions
- Own the folder structure and module boundaries

**Skills Required**: TypeScript, CI/CD, Playwright internals, design patterns (POM, Fixtures)

**Key Files Owned**:
- `playwright.config.ts`
- `fixtures/`
- `hooks/`
- `CLAUDE.md`
- `rules.md`

---

### 🟢 Agent: E2E Test Engineer

**Persona**: QA Engineer focused on user journey and workflow validation  
**Goal**: Write and maintain comprehensive end-to-end test suites.

**Responsibilities**:
- Author E2E tests in `tests/e2e/`
- Create and maintain Page Object Models in `pages/`
- Map user journeys to test scenarios
- Identify and cover critical paths and edge cases
- Collaborate with product/dev on acceptance criteria
- Keep tests green and flake-free

**Skills Required**: Playwright selectors, POM pattern, user journey analysis, cross-browser testing

**Key Files Owned**:
- `tests/e2e/`
- `pages/`
- `data/`

**Test Naming Convention**:
```
[Feature].[Scenario].[ExpectedOutcome].spec.ts
# Example: login.validCredentials.shouldRedirectToDashboard.spec.ts
```

---

### 🟡 Agent: API Test Engineer

**Persona**: QA Engineer focused on backend contract and integration validation  
**Goal**: Validate all API endpoints for correctness, contract adherence, and error handling.

**Responsibilities**:
- Author API tests in `tests/api/`
- Validate request/response schemas
- Test authentication, authorization, and error states
- Set up and tear down API test data
- Integrate API checks within E2E flows when needed
- Monitor API response times for performance baselines

**Skills Required**: REST API concepts, JSON schema validation, Playwright `request` context, OpenAPI/Swagger reading

**Key Files Owned**:
- `tests/api/`
- `helpers/apiClient.ts`

**Example Pattern**:
```typescript
test('GET /users returns 200 with valid schema', async ({ request }) => {
  const response = await request.get('/api/users');
  expect(response.status()).toBe(200);
  const body = await response.json();
  expect(body).toHaveProperty('users');
});
```

---

### 🟠 Agent: Smoke & Regression Gatekeeper

**Persona**: QA Engineer responsible for release quality gates  
**Goal**: Ensure critical functionality is verified before every release.

**Responsibilities**:
- Maintain the smoke test suite in `tests/smoke/`
- Define which tests constitute the regression suite
- Tag tests with `@smoke`, `@regression`, `@critical`
- Coordinate test runs with release management
- Report release readiness based on test results
- Investigate and escalate critical failures

**Skills Required**: Risk-based testing, test prioritization, release management, reporting

**Key Files Owned**:
- `tests/smoke/`
- CI/CD test stage configuration

**Test Tags**:
```typescript
test('Homepage loads successfully @smoke @critical', async ({ page }) => { ... });
test('User can login @smoke @regression', async ({ page }) => { ... });
```

---

### 🔴 Agent: Data & Environment Manager

**Persona**: QA Engineer / DevOps hybrid responsible for test data and environments  
**Goal**: Ensure test data integrity and environment stability for reliable test runs.

**Responsibilities**:
- Manage test data files in `data/`
- Create and maintain data factories and fixtures
- Manage `.env` files and environment configurations
- Set up test databases and seed scripts
- Ensure test isolation (no data leakage between tests)
- Monitor environment health during test runs

**Skills Required**: Data management, environment configuration, database basics, dotenv, CI secrets

**Key Files Owned**:
- `data/`
- `config/`
- `.env.example`

---

### 🤖 Agent: Claude AI (Pair Programmer)

**Persona**: AI assistant embedded in the QA workflow  
**Goal**: Accelerate test authoring, debugging, and framework evolution.

**Responsibilities**:
- Generate test code following framework conventions
- Debug failing tests and suggest fixes
- Refactor and improve existing test code
- Write page objects and helper functions
- Explain Playwright APIs and best practices
- Review code for anti-patterns and flakiness risks
- Draft test plans and scenarios from requirements

**Constraints**:
- Always follow `rules.md` — no exceptions
- Never generate tests with hard waits (`waitForTimeout`)
- Always use POM pattern for UI interactions
- Always prefer `data-testid` selectors when available
- Flag any ambiguous requirements before generating code
- Ask clarifying questions if the test scope is unclear

**Prompt Examples**:
```
"Generate a login page object for this URL: [url]"
"Write a test for the checkout flow covering happy path and payment failure"
"Debug this failing test and explain the root cause: [error]"
"Refactor this test to remove the hard wait"
"What's the best selector strategy for this element: [HTML snippet]"
```

---

## Agent Interaction Model

```
Product/Dev → Requirements → E2E Agent + API Agent
                                  ↓
                            Claude AI (assists)
                                  ↓
                          Framework Architect (reviews)
                                  ↓
                        Smoke/Regression Gatekeeper (gates)
                                  ↓
                         Data/Environment Manager (supports)
```

---

## Escalation Path

| Issue | First Contact | Escalate To |
|-------|--------------|-------------|
| Flaky test | E2E / API Agent | Framework Architect |
| Environment down | Data/Env Manager | DevOps |
| Release blocker | Smoke Gatekeeper | QA Lead |
| Framework bug | Framework Architect | QA Lead |
| AI-generated bad code | Claude AI (re-prompt) | E2E Agent |
