# Phase 1: Foundations — The Engine & The Basics

---

## Module 1.1 — Playwright's Architecture: Browser, Context, Page
*(Covered in full depth live — this is the condensed recap.)*

**Analogy:** Browser = the port (expensive, one process). BrowserContext = an isolated shipping container (cheap, fully isolated cookies/storage). Page = a tab/door on that container (cheapest, shares state with sibling pages in the same context).

**Core mechanics:**
- Playwright talks to browsers over a persistent **WebSocket** using CDP (Chromium) or patched equivalents (Firefox/WebKit) — bidirectional, event-driven.
- Selenium/WebDriver uses synchronous HTTP/JSON round-trips through a separate driver binary (chromedriver/geckodriver) — no persistent channel, no push events.
- Hierarchy: 1 Browser → N BrowserContexts (fully isolated) → N Pages per context (share cookies/storage with siblings).

**Best practices:** one Browser per worker; fresh BrowserContext per test; use `newContext(options)` for viewport/geo/storageState instead of relaunching.
**Pitfalls:** launching a new Browser per test (slow); sharing Page/Context across tests (state leakage); forgetting popups spawn a *new Page* in the *same* context.
**Cost:** Browser launch ~500ms–1.5s | new Context ~20-50ms | new Page ~5-15ms.

**Quiz:** Why isolate at the Context level rather than the Browser level?
**Challenge:** 3 concurrent contexts, each navigating a different URL, screenshot each, graceful per-context failure handling, guaranteed cleanup via `finally`.

---

## Module 1.2 — Installation, Project Setup & Config, Test Runner Internals

**Analogy:** `playwright.config.ts` is your project's constitution — every test inherits its rules (timeouts, browsers, retries) unless it explicitly overrides them.

**Core mechanics:**
- `npm init playwright@latest` scaffolds config + example tests + CI workflow.
- `projects` array in config = a matrix of environments (e.g., chromium/firefox/webkit, or desktop/mobile) — each spec file runs once *per project* automatically.
- Test runner (`@playwright/test`) is built on top of a custom runner (not Jest/Mocha) — it owns fixture resolution, parallelization, retries, and reporting natively.
- Config resolution order: CLI flags > `test.use()` in-file > `project` config > global `use` config.

**Best practices:**
- Keep one `baseURL` in config; use relative paths in `page.goto()`.
- Use `projects` to fan out cross-browser instead of duplicating spec files.
- Set `retries` higher in CI than locally (`process.env.CI ? 2 : 0`).
- Version-pin your Playwright + browsers together (`npx playwright install` after every upgrade).

**Pitfalls:**
- Forgetting `npx playwright install` after a version bump → stale browser binaries, mysterious failures.
- Hardcoding absolute URLs instead of using `baseURL` → breaks when promoting tests across environments.
- Overriding global timeout per-test everywhere instead of tuning it once in config → signals a deeper waiting-strategy problem (see 2.4).

**Code:**
```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: [['html'], ['list']],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
});
```

**Quiz:** What's the precedence order when the same option is set in both `use` (global) and a `project`'s `use`?
**Challenge:** Configure a project matrix that runs desktop Chrome + a mobile Safari emulation (`devices['iPhone 13']`), with CI-only retries and a custom `BASE_URL` read from env.

---

## Module 1.3 — Locators Deep Dive

**Analogy:** A locator isn't "a found element" — it's a *recipe* for finding an element, re-evaluated fresh every single time you act on it. Selenium's `WebElement` is a photograph; Playwright's `Locator` is a live camera feed.

**Core mechanics:**
- `page.locator()` and role/text/testid helpers return a **lazy, re-resolvable handle** — no DOM query happens until an action/assertion is called on it.
- Because it re-queries every time, a locator survives DOM re-renders (React/Vue re-mount) that would make Selenium throw `StaleElementReferenceException`.
- Priority order (per Playwright's own guidance): `getByRole` > `getByText`/`getByLabel` > `getByTestId` > CSS/XPath (escape hatch only).
- Locators are chainable and filterable: `.filter()`, `.and()`, `.or()`, `.nth()`, `.first()`, `.last()`.

**Best practices:**
- Prefer **user-facing, accessible** locators (`getByRole('button', { name: 'Submit' })`) — they double as a lightweight a11y check and are resilient to markup/CSS refactors.
- Use `data-testid` only when no accessible attribute exists.
- Never index into arrays of generic CSS matches (`div.item >> nth=3`) as your primary strategy — it breaks the moment the list order/count changes.

**Pitfalls:**
- Using brittle CSS chains copied from DevTools (`div > div:nth-child(2) > span`) — breaks on any markup change.
- Calling `.textContent()` or `.click()` on a raw `ElementHandle` captured once and reused later — reintroduces the staleness problem Playwright was designed to avoid.
- Overusing XPath `contains(text(), ...)` when `getByText()` does the same thing more readably.

**Code:**
```typescript
// Basic
await page.getByRole('button', { name: 'Sign in' }).click();

// Chained/filtered — production pattern for repeated list items
const row = page.getByRole('row').filter({ hasText: 'Invoice #4521' });
await row.getByRole('button', { name: 'Delete' }).click();
```

**Quiz:** Why doesn't a Playwright `Locator` throw a "stale element" error the way Selenium's `WebElement` does after a re-render?
**Challenge:** Given a table of rows with a "Status" column, write a locator chain that finds the row containing "Pending", then clicks that row's "Approve" button — without using any index-based (`nth`) selection.

---

## Module 1.4 — Actions & Auto-Waiting Mechanics

**Analogy:** Before a delivery driver hands you a package, they check: is this the right door, is it unlocked, are you actually standing there, can you physically take it? Playwright runs the same checklist before every action.

**Core mechanics:** Before executing `click`, `fill`, `check`, etc., Playwright's **actionability checks** verify the target is:
1. **Attached** to the DOM
2. **Visible** (has non-zero bounding box, not `visibility:hidden`)
3. **Stable** (not mid-animation/transition — two consecutive frames with the same bounding box)
4. **Enabled** (not `disabled`)
5. **Receives events** (not obscured by another element on top of it)
6. (For `fill`) **Editable**

All checks are retried against the actionability polling loop until the action's timeout expires — this *is* auto-waiting; it's not a blind `sleep`.

**Best practices:** Trust auto-waiting — don't wrap actions in manual retry loops or `waitForTimeout` "just in case."
**Pitfalls:**
- Clicking through an overlay/spinner that hasn't disappeared yet — auto-wait will correctly time out and tell you *why* (which pitfall applies) if you read the error, not just "TimeoutError."
- Force-clicking (`{ force: true }`) to bypass actionability checks — masks a real bug (e.g., element genuinely obscured) instead of fixing it.

**Code:**
```typescript
// Playwright waits for: attached, visible, stable, enabled, receives-events — automatically
await page.getByRole('button', { name: 'Submit' }).click();

// Anti-pattern — don't do this, it hides real waiting problems
await page.waitForTimeout(2000);
await page.getByRole('button', { name: 'Submit' }).click({ force: true });
```

**Quiz:** Name the 5 actionability checks Playwright runs before a `click()`. What does a timeout on this step actually tell you about your app?
**Challenge:** A button is disabled until a form's checkbox is checked. Write a test that checks the checkbox, then clicks the button — relying entirely on auto-waiting, with zero manual waits or `force: true`.

---

## Module 1.5 — Assertions & Web-First Polling

**Analogy:** A traditional `assert x === y` is a single photograph taken once. Playwright's `expect(locator)` assertions are a **polling video camera** — they keep re-checking the condition until it's true or the timeout expires.

**Core mechanics:**
- `expect(locator).toHaveText(...)`, `.toBeVisible()`, etc. are **retrying/web-first assertions** — internally they poll the DOM state at short intervals until match or timeout.
- Contrast with `expect(await locator.textContent()).toBe(...)` — this snapshots *once*, immediately, with no retry; a legitimate race condition will make this flaky.
- Soft assertions (`expect.soft()`) record failures but let the test continue, collecting *all* failures instead of stopping at the first.

**Best practices:** Always assert on the **locator**, not on a pre-extracted value, so you get retry-ability for free.
**Pitfalls:** `await expect(locator).toBeVisible()` vs `expect(await locator.isVisible()).toBe(true)` — the second is a snapshot with no retry and is a classic source of flaky "it works when I debug it slowly" bugs.

**Code:**
```typescript
// Good — retries internally until true or timeout
await expect(page.getByText('Order confirmed')).toBeVisible();

// Bad — single snapshot, no retry, prone to race conditions
expect(await page.getByText('Order confirmed').isVisible()).toBe(true);

// Soft assertions — collect all failures in one run instead of failing fast
await expect.soft(page.getByTestId('price')).toHaveText('$20.00');
await expect.soft(page.getByTestId('qty')).toHaveText('2');
```

**Quiz:** Why is `expect(locator).toBeVisible()` more reliable under a network-delay scenario than `expect(await locator.isVisible()).toBe(true)`?
**Challenge:** Write a test using `expect.soft` to check three independent fields on a receipt page (price, quantity, total) so that if all three are wrong, the test report shows all three failures in a single run.

---

## Module 1.6 — Navigation & Lifecycle Events

**Analogy:** A page load isn't a single light switch (on/off) — it's several checkpoints in sequence: "HTML downloaded," "DOM parsed," "all sub-resources loaded," "network gone quiet." Picking the wrong checkpoint to wait on is a common source of flakiness.

**Core mechanics:**
- `domcontentloaded`: HTML parsed, DOM tree built — images/stylesheets may still be loading.
- `load`: all resources (images, CSS, iframes) finished loading.
- `networkidle`: no network connections for 500ms — **discouraged by Playwright's own docs** for modern SPAs, since apps with polling/analytics/websockets may *never* go idle.
- `page.goto()` waits for `load` by default; can be tuned via `waitUntil`.
- SPA navigations (client-side routing) often don't fire a full navigation event at all — you should wait for a **specific UI signal** (an element appearing) rather than a lifecycle event.

**Best practices:** For SPAs, wait for a concrete post-navigation signal (`expect(locator).toBeVisible()`) rather than `networkidle`.
**Pitfalls:** Relying on `networkidle` for apps with background polling/websockets — the wait will time out or wait far longer than necessary.

**Code:**
```typescript
// Classic multi-page app
await page.goto('/dashboard', { waitUntil: 'load' });

// SPA — don't chase networkidle; wait for the real signal your test cares about
await page.getByRole('link', { name: 'Dashboard' }).click();
await expect(page.getByRole('heading', { name: 'Welcome back' })).toBeVisible();
```

**Quiz:** Why does Playwright's documentation actively discourage using `networkidle` as a general-purpose waiting strategy for modern web apps?
**Challenge:** Automate a client-side-routed SPA navigation (no full page reload) from a list view to a detail view, asserting on a concrete UI element rather than any navigation lifecycle event.
