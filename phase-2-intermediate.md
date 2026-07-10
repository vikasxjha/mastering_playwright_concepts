# Phase 2: Intermediate — Real Test Engineering

---

## Module 2.1 — Test Isolation & Fixtures

**Analogy:** Fixtures are like a restaurant's mise en place — everything a chef (test) needs is prepped and handed over fresh before they start, and cleared away after, regardless of whether the dish (test) succeeded or burned.

**Core mechanics:**
- Built-in fixtures (`page`, `context`, `browser`, `request`) are created **fresh per test** (or per worker, for `browser`) and torn down automatically — this *is* how Playwright achieves test isolation without you writing setup/teardown boilerplate.
- Custom fixtures are defined via `test.extend<T>({...})` — each fixture is a generator-style function: code before `use()` is setup, code after is teardown, and it only runs if some test actually requests that fixture (lazy).
- Fixtures can depend on other fixtures, forming a dependency graph resolved automatically — this is Playwright's built-in **dependency injection** container (deep dive in Module 4.4).

**Best practices:** Model shared setup (logged-in page, seeded API client, test-data factory) as fixtures, not `beforeEach` blocks — fixtures compose and are reusable across files; `beforeEach` logic isn't.
**Pitfalls:** Putting expensive setup in `beforeEach` when it could be a **worker-scoped** fixture (created once per worker, reused across tests) — needless repeated cost.

**Code:**
```typescript
// fixtures.ts
import { test as base, expect } from '@playwright/test';

type Fixtures = { loggedInPage: import('@playwright/test').Page };

export const test = base.extend<Fixtures>({
  loggedInPage: async ({ page }, use) => {
    await page.goto('/login');
    await page.getByLabel('Username').fill('demo');
    await page.getByLabel('Password').fill('demo123');
    await page.getByRole('button', { name: 'Log in' }).click();
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
    await use(page);          // <-- test runs here
    // teardown after this line runs automatically, even on test failure
  },
});
export { expect };
```

**Quiz:** What determines whether a fixture's setup code even runs for a given test?
**Challenge:** Create a fixture `apiClient` that authenticates against a REST API once per **worker** (not per test) and hands back a pre-authenticated request context to any test that needs it.

---

## Module 2.2 — Hooks, Test Grouping & Annotations

**Core mechanics:**
- `test.describe()` groups related tests and can nest; hooks (`beforeEach`/`afterEach`/`beforeAll`/`afterAll`) scope to their containing describe block.
- `test.step()` wraps a logical chunk of a test for readable trace/report output — doesn't change execution, purely reporting/debugging ergonomics.
- Annotations: `test.skip()`, `test.fixme()` (skipped + flagged as broken/needs fixing), `test.only()` (local-only — never commit), `test.slow()` (triples the timeout).

**Best practices:** Use `test.step()` liberally in long tests — it turns an opaque trace into a readable timeline. Use `fixme` (not a silent `skip`) for known-broken tests so they're tracked, not forgotten.
**Pitfalls:** Committing `test.only()` — silently disables the rest of the suite in CI; guard against it with an ESLint rule (`playwright/no-focused-test`).

**Code:**
```typescript
test.describe('Checkout flow', () => {
  test.beforeEach(async ({ page }) => { await page.goto('/cart'); });

  test('applies a discount code', async ({ page }) => {
    await test.step('add item to cart', async () => {
      await page.getByRole('button', { name: 'Add to cart' }).click();
    });
    await test.step('apply discount', async () => {
      await page.getByLabel('Promo code').fill('SAVE10');
      await page.getByRole('button', { name: 'Apply' }).click();
    });
    await expect(page.getByText('10% discount applied')).toBeVisible();
  });
});
```

**Quiz:** What's the practical difference between `test.skip()` and `test.fixme()`, and why does that difference matter for tracking technical debt?
**Challenge:** Refactor a 40-line single test into 4 `test.step()` blocks, and add a `test.describe` with a shared `beforeEach` that seeds a cart via API instead of UI clicks.

---

## Module 2.3 — Complex UI: iframes, Shadow DOM, Drag-and-Drop, Uploads

**Core mechanics:**
- **iframes**: use `page.frameLocator('iframe selector')` — returns a scoped locator root; Playwright auto-pierces same-origin and cross-origin frames transparently for querying (not for navigation).
- **Shadow DOM**: Playwright's engine pierces open shadow roots **automatically** for standard locators — no special API needed, unlike Selenium which requires manual `shadowRoot` traversal.
- **Drag-and-drop**: `locator.dragTo(target)` for simple cases; for custom drag implementations (canvas-based, HTML5 dataTransfer edge cases) drop to manual `hover()` → `mouse.down()` → `mouse.move()` → `mouse.up()`.
- **File upload**: `locator.setInputFiles(path)` bypasses the OS native file picker dialog entirely — no need to automate the OS dialog.
- **File download**: listen for the `download` event (`page.waitForEvent('download')`) around the triggering action.

**Best practices:** Prefer `setInputFiles` over trying to interact with `<input type=file>` via click+OS dialog (which Playwright can't drive — OS dialogs are outside browser automation's reach).
**Pitfalls:** Trying to `page.locator()` directly into iframe content without `frameLocator` — silently finds nothing since the element lives in a different document context.

**Code:**
```typescript
// iframe
await page.frameLocator('#payment-iframe').getByLabel('Card number').fill('4242424242424242');

// File upload — no OS dialog automation needed
await page.getByLabel('Upload resume').setInputFiles('fixtures/resume.pdf');

// File download
const [download] = await Promise.all([
  page.waitForEvent('download'),
  page.getByRole('button', { name: 'Export CSV' }).click(),
]);
await download.saveAs('downloads/export.csv');

// Drag and drop
await page.getByText('Task A').dragTo(page.getByTestId('in-progress-column'));
```

**Quiz:** Why doesn't Playwright need a special API to interact with elements inside an *open* shadow root, but does need `frameLocator` for iframes?
**Challenge:** Automate uploading a file into a form embedded inside an iframe, then assert a success toast appears in the parent page.

---

## Module 2.4 — Waiting Strategies & Race Conditions

**Analogy:** `waitForTimeout` is setting an alarm clock and hoping the bus arrives by then. `waitForResponse`/`waitForEvent` is waiting at the bus stop and reacting the instant the bus (the actual event) shows up.

**Core mechanics:**
- `page.waitForResponse(urlOrPredicate)` / `waitForRequest()` — resolve the instant a matching network event fires; must be set up (often via `Promise.all`) *before* the triggering action to avoid a race where the response arrives before you start listening.
- `page.waitForEvent('popup'|'download'|'dialog'|...)` — same pattern for browser-level events.
- `locator.waitFor({ state: 'visible'|'hidden'|'attached'|'detached' })` — explicit state wait when auto-waiting on an action isn't the right fit (e.g., waiting for a spinner to *disappear* before proceeding).

**Best practices:** Always pair the wait-setup and the triggering action inside `Promise.all([waitForX(...), triggerAction()])` — never `await` the wait first, then separately trigger the action (classic race condition).
**Pitfalls:** `await page.waitForTimeout(n)` used as a general-purpose "just in case" wait — masks real synchronization issues and makes tests both slow *and* still flaky (n might not be enough on a slower CI runner).

**Code:**
```typescript
// Correct pattern — listener attached BEFORE the action that triggers it
const [response] = await Promise.all([
  page.waitForResponse((res) => res.url().includes('/api/orders') && res.status() === 200),
  page.getByRole('button', { name: 'Place order' }).click(),
]);
const body = await response.json();
expect(body.status).toBe('confirmed');

// Waiting for a spinner to disappear
await page.getByTestId('loading-spinner').waitFor({ state: 'hidden' });
```

**Quiz:** Why must `waitForResponse` be started in the same `Promise.all` as the action that triggers the response, rather than awaited beforehand?
**Challenge:** Automate a "Submit" button that triggers an async API call; assert on the actual response payload (not just a UI success message), handling the case where the API might return a 4xx validation error instead.

---

## Module 2.5 — Authentication Strategies

**Analogy:** Logging in through the UI for every test is like re-keying your house's locks every morning before checking if the mail arrived. `storageState` is handing yourself a spare key once and reusing it.

**Core mechanics:**
- `context.storageState({ path })` serializes cookies + localStorage after a real UI (or API) login; `browser.newContext({ storageState: path })` rehydrates that exact session instantly.
- **Global setup** (`globalSetup` in config, or a `setup` project with `testDependencies`) runs the login *once* before the whole suite, writing the storage state to disk for all tests to reuse.
- Multi-role apps: generate one storageState file per role (admin.json, user.json), select per-project or per-test via `test.use({ storageState: 'admin.json' })`.

**Best practices:** Log in once via UI (or, faster, directly via API call that sets the auth cookie) in setup, not per test.
**Pitfalls:** Re-running full UI login in every test's `beforeEach` — multiplies suite runtime by the number of tests for zero additional coverage value (login flow is presumably already covered by its own dedicated test).

**Code:**
```typescript
// auth.setup.ts — a "setup" project dependency
import { test as setup, expect } from '@playwright/test';

setup('authenticate as admin', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Username').fill('admin');
  await page.getByLabel('Password').fill('adminpass');
  await page.getByRole('button', { name: 'Log in' }).click();
  await expect(page.getByRole('heading', { name: 'Admin panel' })).toBeVisible();
  await page.context().storageState({ path: 'playwright/.auth/admin.json' });
});

// playwright.config.ts (excerpt)
projects: [
  { name: 'setup', testMatch: /auth\.setup\.ts/ },
  { name: 'chromium', use: { storageState: 'playwright/.auth/admin.json' }, dependencies: ['setup'] },
],
```

**Quiz:** What's the performance argument for authenticating once in a `globalSetup`/setup-project versus in every test's `beforeEach`?
**Challenge:** Set up two storage-state files (admin and regular user) and configure two Playwright projects that each pick up the correct role automatically via `dependencies`.

---

## Module 2.6 — Network Interception & Mocking

**Analogy:** `page.route()` puts you in as the postal sorting office between the browser and the real internet — you can inspect, redirect, delay, corrupt, or simply invent the mail (responses) before it reaches its destination (the page).

**Core mechanics:**
- `page.route(urlPattern, handler)` intercepts matching requests; inside the handler you can `route.fulfill()` (return a fake response), `route.continue()` (pass through, optionally modified), or `route.abort()`.
- Enables testing UI states that are hard to reproduce with a real backend: error responses, empty states, slow networks, edge-case payloads.
- HAR (HTTP Archive) recording/replay (`recordHar` / `routeFromHAR`) lets you capture real traffic once and replay it deterministically in tests — useful for third-party APIs you don't control.

**Best practices:** Mock at the network layer (not by stubbing JS functions) — it exercises the real frontend code path against a controlled backend response.
**Pitfalls:** Over-mocking everything, so tests no longer verify real integration — balance E2E tests (real backend) against network-mocked tests (fast, deterministic edge cases) deliberately, don't mock by default everywhere.

**Code:**
```typescript
// Force an error state that's hard to trigger against a real backend
await page.route('**/api/checkout', (route) =>
  route.fulfill({ status: 500, body: JSON.stringify({ error: 'Payment gateway down' }) })
);
await page.getByRole('button', { name: 'Pay now' }).click();
await expect(page.getByText('Payment gateway down')).toBeVisible();

// Modify a real response in-flight
await page.route('**/api/products', async (route) => {
  const response = await route.fetch();
  const json = await response.json();
  json.push({ id: 999, name: 'Injected test product', price: 0 });
  await route.fulfill({ response, json });
});
```

**Quiz:** What's the trade-off between a fully network-mocked test and a full E2E test hitting a real backend?
**Challenge:** Simulate a slow/degraded network for the checkout API (introduce a 3-second delay via `route`) and assert the UI shows a loading state rather than appearing frozen.

---

## Module 2.7 — Visual Regression Testing

**Core mechanics:**
- `expect(page).toHaveScreenshot()` captures a pixel comparison against a stored baseline, using a configurable `maxDiffPixelRatio`/`threshold` for anti-aliasing tolerance.
- Baselines are OS/browser-specific by default (rendering differs across platforms) — CI screenshots should be generated *in* CI (or via Docker) rather than committed from a local machine.
- `mask: [locator]` excludes dynamic regions (timestamps, ads, animations) from the diff instead of failing on expected variance.

**Best practices:** Generate/update baselines in the same environment (Docker image/CI) that will run comparisons — never mix local-Mac-generated baselines with Linux-CI comparisons.
**Pitfalls:** Snapshotting entire pages with live/dynamic content (ads, timestamps, carousels) without masking — guarantees flaky diffs unrelated to real regressions.

**Code:**
```typescript
test('dashboard visual snapshot', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [page.getByTestId('last-updated-timestamp')],
    maxDiffPixelRatio: 0.02,
  });
});
```

**Quiz:** Why must visual baselines be generated in the same OS/environment used for comparison in CI?
**Challenge:** Add a visual test for a page with a live-updating "last synced Xs ago" widget, masking only that widget so the rest of the layout is still fully verified.

---

## Module 2.8 — API Testing with Playwright's `request` Context

**Analogy:** Sometimes you don't need to drive the whole car to check the engine — Playwright's `request` fixture lets you talk straight to the API layer, no browser rendering involved, for setup, teardown, or pure API test suites.

**Core mechanics:**
- `request` fixture (or `request.newContext()` standalone) issues real HTTP calls sharing Playwright's context/cookie machinery — can even share auth state with a UI `page` via the same `storageState`.
- Enables **hybrid tests**: seed data via a fast API call, then verify the *UI* reflects it — far faster and more reliable than seeding through UI forms.
- Also usable as a standalone API test runner — full assertion support (`expect(response).toBeOK()`, JSON body assertions) without ever opening a browser.

**Best practices:** Use API calls for test data setup/teardown; reserve UI automation for what you're actually testing (the UI itself).
**Pitfalls:** Seeding all test data through slow UI form flows when a single API call would do the same job in milliseconds — inflates suite runtime for no added coverage.

**Code:**
```typescript
test('newly created order via API appears in UI order history', async ({ page, request }) => {
  const res = await request.post('/api/orders', { data: { item: 'Widget', qty: 2 } });
  expect(res.ok()).toBeTruthy();
  const { orderId } = await res.json();

  await page.goto('/orders');
  await expect(page.getByText(orderId)).toBeVisible();
});
```

**Quiz:** What's the benefit of seeding test preconditions via `request` rather than via UI interactions?
**Challenge:** Write a hybrid test that creates a resource via API, verifies it in the UI, then deletes it via API in an `afterEach` — leaving zero test-created data behind regardless of test outcome.
