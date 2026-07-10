# Phase 3: Advanced — Scale, Reliability & Debugging

---

## Module 3.1 — Parallelism & Sharding Internals

**Analogy:** Workers are separate factory floors (separate OS processes, each with its own Browser); shards are entirely separate factories (separate machines in CI), each building a fraction of the total product.

**Core mechanics:**
- Each **worker** is an isolated Node.js process with its own Browser instance; tests within a worker run serially, but many workers run concurrently — default worker count ≈ CPU cores / 2.
- Workers are **not** guaranteed to be reused across test *files* the same way — a fresh worker may be spun up if a previous one crashed, but generally one worker processes a queue of files sequentially, providing natural isolation between files without repaying the Browser-launch cost for every file.
- **Sharding** (`--shard=1/4`) splits the *entire test file list* across separate CI machines/jobs — each shard runs a disjoint subset; requires merging reports afterward (Module 5.2).

**Best practices:** Design tests to be independent and stateless so they can run in any worker, any order, any shard, with no shared mutable fixtures across files.
**Pitfalls:** Tests that depend on execution order or on a previous test's leftover state — breaks the moment you increase worker count or shard the suite, since ordering is no longer guaranteed.

**Code:**
```bash
# Run with 4 parallel workers locally
npx playwright test --workers=4

# CI: this machine runs shard 1 of 4 (pair with 3 other CI jobs running 2/4, 3/4, 4/4)
npx playwright test --shard=1/4
```

**Quiz:** What's the structural difference between increasing `--workers` and using `--shard`, and when would you need both?
**Challenge:** Identify (in a hypothetical suite) a test that silently depends on running after another test, and rewrite it to be order-independent using a fixture instead of shared global state.

---

## Module 3.2 — Debugging Mastery

**Analogy:** The Trace Viewer is a flight recorder's black box — after a crash, you don't guess what happened, you scrub through a full timeline of DOM snapshots, network calls, console logs, and actions frame by frame.

**Core mechanics:**
- `trace: 'on'|'retain-on-failure'|'on-first-retry'` in config records a full timeline: DOM snapshots at every step, network requests, console output, and screenshots — viewable via `npx playwright show-trace trace.zip`.
- `--debug` flag / `PWDEBUG=1` launches the **Playwright Inspector** — step through actions one at a time, live-edit locators in the console, see the actionability checks in real time.
- `page.pause()` inserted in code drops you into the Inspector at that exact point, mid-test, with full access to the live page.
- Codegen (`npx playwright codegen`) records your manual browser interactions and emits Playwright code — a good starting point/reference for locators, never a final production artifact as-is.

**Best practices:** Set `trace: 'retain-on-failure'` in CI by default — near-zero cost for passing tests, invaluable for failures. Use Inspector locally for anything non-obvious rather than adding print statements.
**Pitfalls:** Debugging flaky CI failures purely by reading logs instead of pulling the trace artifact — traces show *exactly* what the DOM looked like at the failure point, which logs can't.

**Code:**
```bash
npx playwright test --debug                     # step-through Inspector
PWDEBUG=1 npx playwright test my.spec.ts         # same, via env var
npx playwright show-trace test-results/.../trace.zip   # post-mortem trace analysis
npx playwright codegen https://example.com       # record-and-generate starting code
```

**Quiz:** What specific artifact would you pull from a failed CI run to see the exact DOM state at the moment of failure, and why is that more reliable than console logs alone?
**Challenge:** Reproduce a failing test locally using `--debug`, step through the actionability checks for the failing action, and identify which of the 5 actionability conditions was actually failing.

---

## Module 3.3 — Flakiness Engineering

**Analogy:** Retries are a fever reducer, not a cure — they make the symptom go away long enough to ship, while the underlying infection (race condition, environment dependency) keeps spreading.

**Core mechanics:**
- Playwright's `retries` config re-runs a *failed* test up to N times, keeping the last attempt's result for reporting — useful as a safety net for infra flakiness, dangerous as a substitute for fixing real races.
- Root causes of flakiness, roughly in order of frequency: (1) missing/incorrect waits (Module 2.4), (2) shared/leaked state between tests (Module 2.1/3.1), (3) test-data collisions (two tests mutating the same seeded record), (4) genuine app race conditions the automation is correctly exposing.
- `test.slow()` triples the timeout for a specific test known to be legitimately heavier (e.g., large file upload) — different from papering over a real wait bug.

**Best practices:** Treat a test that only fails "sometimes" as a bug report against either the test *or* the app — investigate via trace before reaching for retries. Track flaky-test rate as a CI health metric, not just pass/fail.
**Pitfalls:** Setting `retries: 3` globally and considering the flakiness "solved" — retries hide the signal that something is genuinely non-deterministic, and the same bug can eventually surface in production.

**Code:**
```typescript
// Legitimate use of test.slow() — known heavy operation, not a masked race
test('uploads a 500MB video file', async ({ page }) => {
  test.slow();
  // ... 
});
```

**Quiz:** List the four common root causes of flaky Playwright tests, ranked by how frequently each shows up in real suites.
**Challenge:** Given a hypothetical test that fails ~1 in 20 runs with a timeout on an element becoming visible, describe the trace-based investigation steps you'd take before touching the retry count.

---

## Module 3.4 — Multi-tab, Multi-context & Multi-origin Scenarios

**Core mechanics:**
- A popup (`target="_blank"`, `window.open`, OAuth redirect) creates a **new Page in the same BrowserContext** — you must explicitly capture it via `page.waitForEvent('popup')` around the triggering click; the original `page` reference never becomes that new tab.
- Multi-origin flows (e.g., app → third-party payment provider → back to app) mean locators/actions must switch between the correct `Page` object at each hop.
- `context.pages()` lists all currently open pages/tabs in a context — useful when the number of resulting tabs is dynamic.

**Best practices:** Always capture the popup reference explicitly via `Promise.all([context.waitForEvent('page'), triggerClick()])` rather than assuming which page object is "active."
**Pitfalls:** Continuing to act on the original `page` variable after a popup/redirect opens a new tab — actions silently target the wrong (now background) tab, or time out waiting for elements that only exist in the new one.

**Code:**
```typescript
const [popup] = await Promise.all([
  context.waitForEvent('page'),
  page.getByRole('link', { name: 'Pay with PayPal' }).click(),
]);
await popup.waitForLoadState();
await popup.getByLabel('Email').fill('buyer@example.com');
// ... complete flow in the popup, which may close itself and return focus to `page`
```

**Quiz:** After a click opens a popup, why can't you keep using the original `page` variable to interact with the new tab's content?
**Challenge:** Automate a flow where clicking "Continue with SSO" opens a popup, you complete a fake login form in that popup, and after the popup closes, assert the original page reflects the logged-in state.

---

## Module 3.5 — Component Testing (Playwright CT)

**Analogy:** E2E tests drive the whole finished car; component tests take a single engine off the line and bench-test it in isolation, at a fraction of the setup cost.

**Core mechanics:**
- `@playwright/experimental-ct-react` (and Vue/Svelte equivalents) mounts a single component in a real browser via a lightweight Vite-based harness — real browser rendering/CSS/events, without a full app server or routing.
- `mount(<Component prop={x} />)` returns a Playwright `Locator`-compatible handle — you interact with it using the exact same API as full-page tests.
- Sits between unit tests (jsdom, no real browser) and full E2E (entire app + backend) — catches real rendering/interaction bugs cheaper than E2E, more realistically than jsdom mocks.

**Best practices:** Use component tests for isolated UI logic (form validation, complex widgets, conditional rendering) where spinning up the whole app is overkill.
**Pitfalls:** Trying to replace all E2E coverage with component tests — component tests can't catch integration/routing/backend-contract issues by definition; they're complementary, not a replacement.

**Code:**
```typescript
// button.test.tsx (Playwright CT for React)
import { test, expect } from '@playwright/experimental-ct-react';
import { PriceButton } from './PriceButton';

test('shows formatted price and fires onClick', async ({ mount }) => {
  let clicked = false;
  const component = await mount(
    <PriceButton price={19.99} onClick={() => (clicked = true)} />
  );
  await expect(component).toHaveText('$19.99');
  await component.click();
  expect(clicked).toBe(true);
});
```

**Quiz:** Where does component testing sit between unit tests and full E2E tests, and what class of bug does it catch that jsdom-based unit tests can't?
**Challenge:** Write a component test for a dropdown component that opens on click, filters options as you type, and fires a callback with the selected value.

---

## Module 3.6 — Accessibility Testing Integration

**Core mechanics:**
- `@axe-core/playwright` injects the axe-core accessibility engine into the page and runs a full WCAG ruleset audit, returning structured violations (rule, severity, affected nodes).
- Complements (doesn't replace) using `getByRole`/`getByLabel` locators throughout your suite — if your *tests* can find elements by role/label, that's already informal evidence the app is somewhat accessible; axe catches the rest (contrast, ARIA misuse, missing alt text) that locator strategy alone won't reveal.

**Best practices:** Run axe scans on key pages/states as part of the regular suite, and fail CI on new "serious"/"critical" violations rather than trying to gate on zero violations across an entire legacy app at once.
**Pitfalls:** Treating an axe scan passing as "the app is accessible" — automated scans catch roughly a third of real-world a11y issues (per axe's own stated coverage); manual review and keyboard-only navigation testing still matter.

**Code:**
```typescript
import AxeBuilder from '@axe-core/playwright';

test('dashboard has no critical a11y violations', async ({ page }) => {
  await page.goto('/dashboard');
  const results = await new AxeBuilder({ page }).analyze();
  const critical = results.violations.filter((v) => v.impact === 'critical');
  expect(critical).toEqual([]);
});
```

**Quiz:** Why is an automated axe-core scan necessary but not sufficient for verifying accessibility?
**Challenge:** Add an axe scan to a page with a known contrast issue, filter violations to only `serious`/`critical` impact, and produce a readable failure message listing the affected selectors.

---

## Module 3.7 — Mobile Emulation & Cross-Browser Strategy

**Core mechanics:**
- `devices['iPhone 13']` etc. (from `playwright`) provide preset viewport, user-agent, and touch-event emulation — this emulates *viewport/UA*, not a real mobile browser engine; WebKit project ≈ real mobile Safari rendering engine, but touch gesture emulation is approximate.
- Firefox and WebKit each have real behavioral differences from Chromium (focus handling, date input rendering, certain CSS edge cases) — cross-browser runs exist specifically to catch these, not just as a checkbox exercise.
- `hasTouch`, `isMobile` context options control touch-event dispatch and viewport meta behavior independent of the full device preset.

**Best practices:** Run your critical-path suite across all three engines in CI; reserve full device-preset matrices (many phone/tablet sizes) for a smaller, targeted responsive-design suite rather than the entire regression suite.
**Pitfalls:** Assuming a Chromium-only green suite means "cross-browser compatible" — Safari/WebKit-specific bugs (a huge share of real-world mobile traffic) are invisible until you actually run against the `webkit` project.

**Code:**
```typescript
// playwright.config.ts (excerpt)
projects: [
  { name: 'Mobile Safari', use: { ...devices['iPhone 13'] } },
  { name: 'Mobile Chrome', use: { ...devices['Pixel 7'] } },
  { name: 'Desktop Firefox', use: { ...devices['Desktop Firefox'] } },
],
```

**Quiz:** Why doesn't a fully green Chromium-only test suite guarantee cross-browser correctness, particularly for mobile traffic?
**Challenge:** Add a project for `iPhone 13` emulation, and write a test that asserts a mobile-only hamburger menu (hidden on desktop viewports) is visible and functional.
