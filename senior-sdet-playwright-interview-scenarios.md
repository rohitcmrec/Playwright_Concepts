# Senior SDET -- Playwright Scenario-Based Interview Problems

> A reference guide containing five challenging Playwright
> scenario-based interview questions, with explanations, root causes,
> and practical solutions.

------------------------------------------------------------------------

## 1. Missing `await` with `textContent()`

### Question

A test does:

``` typescript
const text = page.locator('.total').textContent()
```

without `await` and later asserts on `text`. It passes sometimes.

**Explain what `text` holds, why it intermittently "works", and how the
JS event loop produces that non-determinism.**

### Answer

`page.locator('.total').textContent()` is an **asynchronous Playwright
API**. It returns a `Promise`, not the actual text value.

Without `await`, the variable contains a Promise:

``` typescript
const text = page.locator('.total').textContent();
// text is a Promise, not the string "$100"
```

To get the resolved text, you must await it:

``` typescript
const text = await page.locator('.total').textContent();

expect(text).toBe('$100');
```

### Why does it appear to work sometimes?

A Promise is a JavaScript object. If an assertion only checks whether
the variable exists or is truthy, the test can pass because the Promise
object itself is truthy:

``` typescript
const text = page.locator('.total').textContent();

expect(text).toBeDefined(); // Passes because text is a Promise
```

However, this does **not** mean the asynchronous operation has
completed.

The JavaScript event loop schedules asynchronous work separately from
the current synchronous execution. If the test continues without waiting
for the Promise, later code may execute before the locator operation has
resolved.

This creates timing-dependent behavior when other asynchronous work,
assertions, or browser/test teardown happen around the same time.

### Key interview point

> **`await` is not optional when you need the result of an asynchronous
> Playwright API.**

The reliable pattern is:

``` typescript
const text = await page.locator('.total').textContent();
expect(text).toBe('$100');
```

Even better, when possible, prefer Playwright's web-first assertions
because Playwright will automatically wait for the expected UI state:

``` typescript
await expect(page.locator('.total')).toHaveText('$100');
```

------------------------------------------------------------------------

## 2. `page.evaluate()` and the Browser/Node.js Boundary

### Question

You call:

``` typescript
page.evaluate(() => fetch('/api/x'))
```

and the test continues before the fetch resolves.

**Explain the boundary between the Node-side Playwright context and the
browser JS engine, and how to make the test wait on that in-page async
work correctly.**

### Answer

Playwright tests normally execute in the **Node.js process**, while the
function passed to `page.evaluate()` executes inside the **browser
page's JavaScript environment**.

Think of it as two JavaScript environments:

``` text
Playwright Test / Node.js
        |
        | page.evaluate()
        v
Browser JavaScript Engine
        |
        | fetch('/api/x')
        v
     Network
```

The important point is that `fetch()` in the browser is asynchronous and
returns a Promise.

### Problem

If you start the fetch but don't return/await it:

``` typescript
await page.evaluate(() => {
  fetch('/api/x');
});
```

the callback itself returns `undefined`.

From Playwright's perspective, the `evaluate()` call is complete as soon
as the callback finishes. Playwright therefore has no Promise to wait
for.

### Correct approach

Return the Promise from the browser context and await the result from
Node:

``` typescript
const data = await page.evaluate(async () => {
  const response = await fetch('/api/x');
  return response.json();
});
```

Or:

``` typescript
const data = await page.evaluate(() =>
  fetch('/api/x').then(response => response.json())
);
```

Now Playwright waits for the Promise returned by the browser-side
function.

### Key interview point

> **`page.evaluate()` bridges Node.js and the browser. If the
> browser-side operation is asynchronous, return/await its Promise so
> Playwright can wait for it.**

Also remember that Playwright provides its own request APIs when the
goal is to test an API directly:

``` typescript
const response = await request.get('/api/x');
```

Use `page.evaluate()` when you specifically need the request to
originate from the page/browser context---for example, when browser
cookies, page JavaScript, CORS behavior, or browser context are
relevant.

------------------------------------------------------------------------

## 3. Brittle CSS Selectors vs. User-Facing Locators

### Question

Your team has 400 tests using selectors such as:

``` typescript
page.locator('#app > div > div:nth-child(3)')
```

and they break every redesign.

**Make the case for user-facing locators to a skeptical lead, and
describe a migration strategy that does not require a 400-test rewrite
in one shot.**

### Answer

Deep structural CSS selectors are tightly coupled to the implementation
details of the DOM.

For example:

``` typescript
page.locator('#app > div > div:nth-child(3)')
```

can break if a developer:

-   Adds a wrapper `<div>`
-   Changes the DOM hierarchy
-   Reorders elements
-   Introduces a new component
-   Changes styling structure

The test may fail even though the user-facing behavior has not changed.

### Prefer user-facing locators

Playwright recommends locators that resemble how users interact with the
application.

Examples:

``` typescript
page.getByRole('button', { name: 'Submit' });
page.getByRole('textbox', { name: 'Email' });
page.getByText('Order successful');
page.getByLabel('Password');
```

For example:

``` typescript
await page.getByRole('button', { name: 'Submit' }).click();
```

This communicates the intent of the test much better than:

``` typescript
await page.locator('#app > div > div:nth-child(3)').click();
```

### Why this is valuable

User-facing locators provide:

1.  **Better readability** -- the test describes what the user does.
2.  **Better resilience** -- internal DOM changes are less likely to
    break the test.
3.  **Better accessibility alignment** -- `getByRole()` uses accessible
    roles and names.
4.  **Better maintenance** -- failures are easier to understand.
5.  **Better collaboration** -- QA, developers, and product people can
    understand the test intent.

### Migration strategy for 400 tests

Do **not** attempt a 400-test rewrite in one sprint.

Use an incremental migration.

#### Step 1 -- Establish a rule for new tests

All new tests should use preferred locators:

``` typescript
getByRole()
getByLabel()
getByText()
getByPlaceholder()
```

Use CSS/test IDs only when a user-facing locator is inappropriate.

#### Step 2 -- Refactor shared Page Object Models first

If multiple tests use the same Page Object Model, improving the locator
there can fix many tests at once.

For example:

``` typescript
class CheckoutPage {
  submitButton = this.page.getByRole('button', { name: 'Submit order' });
}
```

Instead of:

``` typescript
submitButton = this.page.locator('#checkout > div:nth-child(4) button');
```

#### Step 3 -- Opportunistic migration

When an existing test is being modified for a feature change or bug fix,
migrate its brittle selectors at the same time.

This avoids creating a huge dedicated rewrite project.

#### Step 4 -- Track technical debt

Track remaining brittle selectors and gradually reduce them.

### Key interview point

> **The goal isn't "never use CSS." The goal is to choose the most
> stable locator that expresses user intent, and migrate incrementally
> rather than creating a risky 400-test rewrite.**

------------------------------------------------------------------------

## 4. Avoiding Repeated UI Login with `storageState`

### Question

Two suites need a logged-in admin user. One uses `beforeEach` to log in
via UI; it is slow and occasionally flakes on the login page.

**Design a fixture-based approach using `storageState` that eliminates
the per-test UI login, and explain where the state is created and how it
is scoped.**

**Hint:** A setup project/dependency that authenticates once and writes
storage state, consumed via `use`.

### Answer

Logging in through the UI before every test is expensive and adds
unnecessary failure points.

Instead, authenticate once in a **setup project**, save the
authenticated browser state, and reuse that state in dependent tests.

`storageState` can preserve authentication-related browser state such as
cookies and local storage.

### Step 1 -- Create an authentication setup

Example:

``` typescript
// auth.setup.ts

import { test as setup, expect } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('Username').fill(process.env.ADMIN_USER!);
  await page.getByLabel('Password').fill(process.env.ADMIN_PASSWORD!);
  await page.getByRole('button', { name: 'Login' }).click();

  await expect(page.getByText('Dashboard')).toBeVisible();

  await page.context().storageState({
    path: '.auth/admin.json',
  });
});
```

The setup test logs in once and writes the authentication state to:

``` text
.auth/admin.json
```

### Step 2 -- Configure a setup project

``` typescript
// playwright.config.ts

import { defineConfig } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },

    {
      name: 'chromium',
      dependencies: ['setup'],
      use: {
        storageState: '.auth/admin.json',
      },
    },
  ],
});
```

The dependency means:

``` text
setup
  |
  | creates authenticated state
  v
chromium tests
  |
  | consume storageState
  v
Already logged in
```

### What happens during execution?

1.  Playwright runs the `setup` project.
2.  The setup test performs the UI login.
3.  Playwright saves the browser storage state.
4.  The dependent project starts.
5.  Each test context is initialized using the saved state.
6.  Tests start authenticated without navigating through the login UI.

### Important scoping consideration

The saved state is reused as a starting point for each test's browser
context. The browser context itself is still isolated per test by
Playwright.

For parallel workers, be careful if tests modify server-side account
state. A single shared authenticated account may still cause data
collisions even though authentication itself is solved.

If tests require independent users or mutable account state, create
separate authentication states/accounts per worker or use appropriate
data isolation.

### Key interview point

> **Authentication setup and test data isolation are separate concerns.
> `storageState` removes repeated UI login, but it does not
> automatically isolate server-side data.**

------------------------------------------------------------------------

## 5. CI Parallelization and Data Collisions

### Question

A suite passes locally with one worker but fails in CI with
`fullyParallel` and 8 workers, with intermittent data collisions.

**Diagnose the likely cause and give two independent fixes at different
layers (test design vs. data isolation).**

### Answer

The most likely cause is **shared mutable test data**.

With one worker, tests may execute sequentially:

``` text
Test A -> creates user -> uses user -> deletes user
Test B -> creates user -> uses user -> deletes user
```

With 8 workers, tests can execute concurrently:

``` text
Worker 1 -> Test A -> creates "John Doe"
Worker 2 -> Test B -> creates "John Doe"
Worker 3 -> Test C -> updates "John Doe"
...
```

This can produce:

-   Duplicate key errors
-   Tests reading another test's data
-   Records being deleted unexpectedly
-   Incorrect assertions
-   Race conditions
-   Intermittent failures

The reason it appears only in CI is that parallel execution exposes the
hidden dependency between tests.

------------------------------------------------------------------------

### Fix 1 -- Test design: Generate unique test data

Avoid hardcoded data shared across tests.

Instead of:

``` typescript
const email = 'testuser@example.com';
```

generate unique data:

``` typescript
const uniqueEmail = `testuser-${Date.now()}@example.com`;
```

An even stronger approach is UUID-based data:

``` typescript
import { randomUUID } from 'crypto';

const uniqueEmail = `testuser-${randomUUID()}@example.com`;
```

For example:

``` typescript
const user = {
  name: `Test User ${randomUUID()}`,
  email: `test-${randomUUID()}@example.com`,
};
```

Now each test creates a different record.

### Fix 2 -- Data isolation: Allocate data/accounts per worker

For tests that need pre-created accounts or other controlled resources,
partition them by Playwright worker.

Conceptually:

``` typescript
const testUser = authUsers[testInfo.workerIndex];
```

For example:

``` typescript
const authUsers = [
  { username: 'worker0-admin', password: '...' },
  { username: 'worker1-admin', password: '...' },
  { username: 'worker2-admin', password: '...' },
  // ...
];

const testUser = authUsers[testInfo.workerIndex];
```

Each worker gets its own account/resource pool.

Another robust strategy is to create isolated test data as part of a
worker-scoped fixture:

``` typescript
import { test as base } from '@playwright/test';

export const test = base.extend<{
  testUser: { email: string };
}>({
  testUser: [async ({}, use, testInfo) => {
    const email =
      `worker-${testInfo.workerIndex}-${crypto.randomUUID()}@example.com`;

    // Create isolated user/resource here.

    await use({ email });

    // Clean up worker-specific data here.
  }, { scope: 'worker' }],
});
```

### Why two fixes?

These solve the problem at different layers:

  -----------------------------------------------------------------------
  Layer                   Fix                     Purpose
  ----------------------- ----------------------- -----------------------
  Test design             Generate unique data    Prevent tests from
                                                  targeting the same
                                                  records

  Data isolation          Per-worker              Prevent workers from
                          accounts/resources      sharing mutable
                                                  server-side state
  -----------------------------------------------------------------------

### Key interview point

> **Parallelization doesn't usually create the bug; it exposes an
> existing lack of test isolation.**

A good SDET should make tests independently executable regardless of
worker count.

------------------------------------------------------------------------

# Quick Interview Revision

  --------------------------------------------------------------------------------
  \#                Scenario                   Core Problem      Best Concept
  ----------------- -------------------------- ----------------- -----------------
  1                 Missing `await`            Promise           `await`, Promise,
                                               vs. resolved      event loop
                                               value             

  2                 `page.evaluate(fetch())`   Node/browser      Return/await
                                               async boundary    browser Promise

  3                 Brittle CSS selectors      DOM               User-facing
                                               implementation    locators
                                               coupling          

  4                 UI login in `beforeEach`   Slow/flaky        Setup project +
                                               repeated          `storageState`
                                               authentication    

  5                 CI data collisions         Shared mutable    Unique data +
                                               test data         isolation
  --------------------------------------------------------------------------------

------------------------------------------------------------------------

# Senior SDET Talking Points

When answering these scenarios in an interview, emphasize these broader
engineering principles:

### 1. Understand asynchronous boundaries

Know exactly where your code is executing:

``` text
Node.js / Playwright
        |
        | evaluate()
        v
Browser JavaScript
        |
        | network / DOM / timers
        v
Browser environment
```

Always know which Promise you are waiting for.

### 2. Prefer intent over implementation

A good test describes user behavior:

``` typescript
getByRole('button', { name: 'Submit' })
```

rather than implementation details:

``` typescript
#app > div > div:nth-child(3)
```

### 3. Separate authentication from test data

`storageState` solves repeated authentication.

It does **not** solve:

-   Shared database records
-   Concurrent updates
-   Shared accounts with mutable state
-   Test cleanup conflicts

### 4. Design for parallel execution

A high-quality test should work with:

``` text
1 worker
4 workers
8 workers
fullyParallel
```

without changing its behavior.

### 5. Treat flakiness as a symptom

Don't simply add:

``` typescript
await page.waitForTimeout(2000);
```

Instead identify the actual synchronization or isolation problem.

Prefer:

``` typescript
await expect(locator).toBeVisible();
await expect(locator).toHaveText('...');
await page.waitForResponse(...);
```

when those conditions represent the real synchronization point.

------------------------------------------------------------------------

## One-Line Answers for Interviews

**Q1 -- Why is missing `await` a problem?**

> Because Playwright returns a Promise; without `await`, the variable
> contains the Promise rather than its resolved value, so assertions can
> execute against the wrong thing.

**Q2 -- Why doesn't `fetch()` inside `evaluate()` automatically block
the test?**

> Because the callback runs in the browser context, and Playwright can
> only wait for the browser-side async operation if the Promise is
> returned from `evaluate()`.

**Q3 -- Why prefer `getByRole()` over deep CSS selectors?**

> It expresses user intent and is less coupled to the application's DOM
> structure.

**Q4 -- Why use `storageState`?**

> Authenticate once during setup and reuse the saved browser state so
> every test doesn't need to repeat a slow and flaky UI login.

**Q5 -- Why does a test pass locally but fail with 8 workers?**

> Parallel workers expose shared mutable test data; make test data
> unique and/or isolate accounts/resources per worker.
