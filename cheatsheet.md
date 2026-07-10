# Playwright Quick-Reference Cheatsheet

Keep this open while coding — it's the "how do I..." lookup companion to the phase files.

## Locators (priority order — top preferred)
```typescript
page.getByRole('button', { name: 'Submit' })     // accessible, resilient — prefer this
page.getByText('Order confirmed')
page.getByLabel('Email')
page.getByPlaceholder('Search...')
page.getByAltText('Company logo')
page.getByTitle('Close')
page.getByTestId('submit-btn')                    // when no accessible attribute exists
page.locator('.css-selector')                     // escape hatch only
page.locator('xpath=//div[@id="x"]')               // last resort

// Chaining / filtering
locator.filter({ hasText: 'foo' })
locator.filter({ has: page.locator('.icon') })
locator.and(other) / locator.or(other)
locator.first() / .last() / .nth(2)
page.frameLocator('#iframe-id').getByRole(...)     // into an iframe
```

## Actions
```typescript
await locator.click();
await locator.dblclick();
await locator.fill('text');            // clears then types
await locator.type('text');            // types char by char (rarely needed over fill)
await locator.check(); / .uncheck();
await locator.selectOption('value');
await locator.hover();
await locator.dragTo(targetLocator);
await locator.setInputFiles('path/to/file.pdf');
await locator.press('Enter');
await locator.focus();
await locator.click({ force: true });  // bypass actionability — avoid unless justified
```

## Assertions (web-first, auto-retrying)
```typescript
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeEnabled();  / .toBeDisabled();
await expect(locator).toHaveText('exact text');
await expect(locator).toContainText('partial');
await expect(locator).toHaveValue('input value');
await expect(locator).toHaveAttribute('href', '/home');
await expect(locator).toHaveCount(3);
await expect(page).toHaveTitle(/Dashboard/);
await expect(page).toHaveURL(/\/orders\/\d+/);
await expect(page).toHaveScreenshot('name.png', { mask: [locator] });
expect.soft(locator).toHaveText('...');           // records but doesn't stop test
```

## Waiting
```typescript
await locator.waitFor({ state: 'visible' | 'hidden' | 'attached' | 'detached' });
await page.waitForURL('**/dashboard');
await page.waitForLoadState('load' | 'domcontentloaded' | 'networkidle');
const [response] = await Promise.all([
  page.waitForResponse((res) => res.url().includes('/api/x')),
  triggerAction(),
]);
const [popup] = await Promise.all([context.waitForEvent('page'), triggerClick()]);
```

## Network
```typescript
await page.route('**/api/x', (route) => route.fulfill({ status: 200, json: {...} }));
await page.route('**/api/x', (route) => route.abort());
await page.route('**/api/x', async (route) => {
  const res = await route.fetch();
  await route.fulfill({ response: res, json: { ...await res.json(), extra: true } });
});
await page.routeFromHAR('trace.har', { url: '**/api/**' });
await page.unroute('**/api/x');
```

## Auth / Storage State
```typescript
await context.storageState({ path: 'auth.json' });
const context = await browser.newContext({ storageState: 'auth.json' });
test.use({ storageState: 'playwright/.auth/admin.json' });
```

## Fixtures
```typescript
export const test = base.extend<{ myFixture: Foo }>({
  myFixture: async ({ page }, use) => {
    const foo = await setup();
    await use(foo);
    await teardown(foo);
  },
});
// Worker-scoped:
myFixture: [async ({}, use) => { await use(x); }, { scope: 'worker' }],
```

## Test Organization
```typescript
test.describe('group', () => { ... });
test.beforeEach(async ({ page }) => { ... });
test.afterEach(async ({ page }) => { ... });
test.beforeAll(async () => { ... });
test.step('readable step name', async () => { ... });
test.skip(condition, 'reason');
test.fixme('broken test', async () => { ... });
test.slow();  // triples timeout
test('name', { tag: '@smoke' }, async ({ page }) => { ... });
```

## CLI Commands
```bash
npx playwright test                          # run all tests
npx playwright test my.spec.ts               # run one file
npx playwright test -g "test name"           # run by title match
npx playwright test --grep @smoke            # run by tag
npx playwright test --headed                 # visible browser
npx playwright test --debug                  # Inspector step-through
npx playwright test --workers=4
npx playwright test --shard=1/4
npx playwright codegen https://example.com   # record actions -> code
npx playwright show-report
npx playwright show-trace trace.zip
npx playwright merge-reports --reporter=html ./blob-reports
npx playwright install --with-deps           # install browsers + OS deps
```

## Config Essentials (`playwright.config.ts`)
```typescript
export default defineConfig({
  testDir: './tests',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: [['html'], ['blob']],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    { name: 'chromium', use: { ...devices['Desktop Chrome'] }, dependencies: ['setup'] },
  ],
});
```

## Design Pattern Cheat-Sheet (Phase 4)
| Pattern | Use it when | Avoid it when |
|---|---|---|
| Page Object (composition) | Standard multi-page app, shared UI chrome | N/A — near-universal baseline |
| Screenplay | Large enterprise app, workflows span many pages/roles | Small app — ceremony outweighs benefit |
| Fixture DI | Always — Playwright's native model | N/A |
| Factory | Producing ready-made domain/test objects | N/A |
| Builder | Test data with many optional fields | Simple flat objects — overkill |
| Strategy | Env/role-specific behavior (auth, notifications) | Only one implementation ever exists |
| Facade | Multi-step workflow test would otherwise inline | Single simple action — unnecessary wrapper |
| Decorator | A few known-flaky things need retry/logging | Applied blanket to everything — adds noise |
