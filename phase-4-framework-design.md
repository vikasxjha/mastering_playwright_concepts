# Phase 4: Automation Framework Design & Engineering Principles

This phase is the heart of "how senior SDETs think" — less about Playwright's API surface, more about software design applied to test code.

---

## Module 4.1 — Software Design Principles for Test Automation

**Analogy:** Untamed test code is a junk drawer — everything works until you need to find one specific thing fast, or change one thing without breaking three others.

**Core mechanics — SOLID applied to test code:**
- **S**ingle Responsibility: a Page Object method does one UI action; a test asserts one behavior; a fixture provides one capability. Splitting "login and also seed cart data" into two responsibilities.
- **O**pen/Closed: extend behavior via composition/fixtures rather than editing a shared base class every time a new page needs slightly different setup.
- **L**iskov Substitution: if you have a `BasePage` and subclasses, any subclass must be usable anywhere the base is expected — a subclass that throws on a method the base guarantees violates this (common with over-eager inheritance in POM).
- **I**nterface Segregation: don't force every Page Object to implement methods it doesn't need (e.g., a "logout" method on a page object for a public marketing page).
- **D**ependency Inversion: tests should depend on abstractions (a `LoginPage` interface/contract) not concrete low-level detail (raw selectors scattered inline) — this is precisely what Playwright's fixture DI system encourages (Module 4.4).

**Why it matters:** violating these at small scale is invisible; at 500+ specs, an untamed framework becomes unmaintainable — a single UI change cascades into dozens of broken tests because logic wasn't isolated.

**Best practices:** Separate three concerns explicitly: **test logic** (what to verify), **UI interaction logic** (how to click/fill), **test data** (what values to use). Never inline all three in one test.
**Pitfalls:** "God" Page Objects with 80 methods and inheritance chains 4 levels deep — violates SRP and ISP simultaneously, becomes untestable itself.

**Quiz:** Give an example of an SRP violation you might see in a hastily-written Page Object class, and explain how you'd split it.
**Challenge:** Take a single test that logs in, navigates, fills a form, and asserts a result — refactor it into three clearly separated layers (data, UI interaction, assertion) without using any specific pattern yet (that comes next).

---

## Module 4.2 — Page Object Model (POM)

**Analogy:** A Page Object is a translator standing between your test's business intent ("log in as admin") and the messy reality of DOM selectors — the test speaks business language; the Page Object speaks DOM.

**Core mechanics:**
- Classic POM: one class per page/screen, encapsulating its locators and the actions available on it; tests call methods (`loginPage.login(user, pass)`), never touch selectors directly.
- **Limitation at scale:** deep inheritance hierarchies (`BasePage` → `AuthenticatedPage` → `DashboardPage`) become brittle — a shared method added to `BasePage` for one subclass's benefit leaks into all others (LSP violation, Module 4.1).
- **Evolution:** favor **composition over inheritance** — a `DashboardPage` *has a* `NavBarComponent` and a `SidebarComponent` rather than inheriting a mega base class. Each component is independently testable/reusable across pages that share that UI fragment.

**Best practices:** Component-composition POM for real apps with shared UI chrome (nav bars, modals, tables) — write one `DataTableComponent` used by every page containing a table, instead of duplicating table-interaction methods per page class.
**Pitfalls:** Encoding assertions *inside* Page Object methods — mixes the "how to interact" layer with the "what to verify" layer, making Page Objects untestable in isolation and coupling every page change to every test's expectations.

**Code:**
```typescript
// Composition-based POM — not deep inheritance
class DataTableComponent {
  constructor(private root: Locator) {}
  row(text: string) { return this.root.getByRole('row').filter({ hasText: text }); }
  async sortBy(column: string) { await this.root.getByRole('columnheader', { name: column }).click(); }
}

class OrdersPage {
  readonly table: DataTableComponent;
  constructor(private page: Page) {
    this.table = new DataTableComponent(page.getByTestId('orders-table'));
  }
  async goto() { await this.page.goto('/orders'); }
  async approveOrder(orderId: string) {
    await this.table.row(orderId).getByRole('button', { name: 'Approve' }).click();
  }
}

// Test — reads like business intent, no selectors visible
test('approving an order updates its status', async ({ page }) => {
  const orders = new OrdersPage(page);
  await orders.goto();
  await orders.approveOrder('INV-4521');
  await expect(orders.table.row('INV-4521')).toContainText('Approved');
});
```

**Quiz:** Why does deep Page Object inheritance tend to violate the Liskov Substitution Principle in real codebases?
**Challenge:** Refactor a `DashboardPage` and `SettingsPage` that both contain a nav bar with duplicated locator logic into a shared `NavBarComponent` used by composition in both.

---

## Module 4.3 — Screenplay Pattern

**Analogy:** POM asks "what can this *page* do?" Screenplay asks "what can this *actor* (user) do, using abilities they carry with them, regardless of which page they're currently on?" It shifts the mental model from page-centric to user-centric.

**Core mechanics — core vocabulary:**
- **Actor**: represents a user/persona (`Actor.named('Admin')`), carrying **Abilities** (e.g., `BrowseTheWeb` — wraps a Playwright `Page`).
- **Task**: a high-level, reusable business action composed of lower-level Interactions (`Login.withCredentials(user, pass)`).
- **Interaction**: the smallest unit — a single click/fill (`Click.on(loginButton)`).
- **Question**: a way to retrieve state for assertions, decoupled from *how* it's fetched (`Text.of(errorBanner)`).

This decouples "what the user is trying to do" from "which page they happen to be on," which scales better than POM when flows span many pages/components (e.g., a checkout flow touching cart, shipping, payment, and confirmation pages) — one Task can orchestrate across all of them without the test needing to know page boundaries at all.

**Best practices:** Reach for Screenplay in large, workflow-heavy enterprise apps where the same business action (e.g., "submit a leave request") appears across many contexts/roles. For simpler, mostly-independent-page apps, POM (or fixture composition) is usually sufficient and has a lower learning curve.
**Pitfalls:** Adopting Screenplay's full ceremony (Actors/Abilities/Tasks/Interactions/Questions) for a small app — the abstraction overhead outweighs the benefit; this is over-engineering for the problem size.

**Code (conceptual — using `@serenity-js/playwright` idioms):**
```typescript
const admin = Actor.named('Admin').whoCan(BrowseTheWebWithPlaywright.using(page));

await admin.attemptsTo(
  Navigate.to('/login'),
  Login.withCredentials('admin', 'adminpass'),
);

const heading = await admin.answer(Text.of(dashboardHeading));
expect(heading).toEqual('Admin panel');
```

**Quiz:** What specific scaling problem does Screenplay solve that classic POM struggles with, and what's the cost of adopting it?
**Challenge:** Describe (in pseudocode, no need for a real library) a `PlaceOrder` Task composed of at least 3 lower-level Interactions spanning 2 different pages, and one Question used to verify the outcome.

---

## Module 4.4 — Fixture-Based Architecture as Dependency Injection

**Analogy:** Traditional base-class inheritance hands every test the *entire* toolbox whether it needs it or not. Playwright's fixture system is a vending machine — a test declares exactly which tools (fixtures) it needs as parameters, and only those get built, lazily, in the right order.

**Core mechanics:**
- `test.extend<T>({...})` fixtures form a **dependency graph**: each fixture can request other fixtures as its own dependencies, and Playwright resolves and instantiates them in the correct order automatically — this is textbook **Dependency Injection**, native to the test runner, no external DI framework needed.
- Fixtures are **lazy**: a fixture only executes its setup if some test in the current run actually destructures it as a parameter — unused fixtures cost nothing.
- Scope control: fixtures declared `{ scope: 'worker' }` are built once per worker process and shared across all tests in that worker (e.g., one authenticated API client reused across many tests); default scope is per-test.
- This directly replaces base-class inheritance patterns from other frameworks (e.g., a Java Selenium `BaseTest` class) — composition of fixtures instead of a rigid class hierarchy (Dependency Inversion, Module 4.1).

**Best practices:** Model every piece of shared capability (auth, seeded data, an API client, a feature-flag override) as its own composable fixture rather than a monolithic base class or a growing `beforeEach`.
**Pitfalls:** Recreating "base class" thinking inside fixtures — e.g., one giant `appFixture` that sets up everything for every test regardless of need, defeating the laziness benefit and coupling unrelated concerns together again.

**Code:**
```typescript
type Fixtures = {
  apiClient: ApiClient;       // worker-scoped — built once, reused
  seededOrder: Order;         // test-scoped — depends on apiClient
};

export const test = base.extend<Fixtures>({
  apiClient: [async ({}, use) => {
    const client = await ApiClient.authenticate(process.env.API_TOKEN!);
    await use(client);
  }, { scope: 'worker' }],

  seededOrder: async ({ apiClient }, use) => {           // depends on apiClient — resolved automatically
    const order = await apiClient.createOrder({ item: 'Widget' });
    await use(order);
    await apiClient.deleteOrder(order.id);               // teardown, always runs
  },
});

test('order appears in history', async ({ page, seededOrder }) => {
  await page.goto(`/orders/${seededOrder.id}`);
  await expect(page.getByText(seededOrder.item)).toBeVisible();
});
```

**Quiz:** How does fixture scoping (`worker` vs. default per-test) let you balance test isolation against setup cost?
**Challenge:** Design a fixture graph with 3 fixtures: a worker-scoped `db` connection, a test-scoped `testUser` (depends on `db`), and a test-scoped `authenticatedPage` (depends on both `testUser` and `page`) — write the `test.extend` block wiring all three together.

---

## Module 4.5 — Creational Patterns in Automation

**Core mechanics:**
- **Factory Pattern**: a function/class that produces configured objects (test data, Page Objects) so tests never call constructors directly with magic values — e.g., `UserFactory.createAdmin()` vs. `new User('admin', 'x', true, false, ...)`.
- **Builder Pattern**: fluent, step-by-step construction of complex test data where most fields have sensible defaults and only a few need overriding per test — avoids constructors with 10+ positional parameters.
- **Singleton pitfalls**: a single shared Browser/Page "singleton" reused across an entire suite for "speed" reintroduces the state-leakage problem from Module 1.1/2.1 — Playwright's worker-scoped Browser fixture already gives you the *safe* version of this idea (shared expensive resource, still test-isolated at the Context level).

**Best practices:** Use Builders for test data with many optional fields; use Factories for producing ready-made domain objects/Page Objects with sensible defaults, overridable per test.
**Pitfalls:** A hand-rolled "global singleton page" module (`export const sharedPage = ...`) to dodge fixture setup cost — reintroduces cross-test coupling that fixtures were designed to eliminate.

**Code:**
```typescript
// Builder — fluent, defaults + selective overrides
class OrderBuilder {
  private order = { item: 'Default Widget', qty: 1, expedited: false };
  withItem(item: string) { this.order.item = item; return this; }
  withQty(qty: number) { this.order.qty = qty; return this; }
  expedited() { this.order.expedited = true; return this; }
  build() { return { ...this.order }; }
}
const rushOrder = new OrderBuilder().withItem('Gadget').withQty(3).expedited().build();

// Factory — ready-made domain objects with sane defaults
const UserFactory = {
  admin: () => ({ username: 'admin', role: 'ADMIN', password: 'adminpass' }),
  guest: () => ({ username: `guest_${Math.random().toString(36).slice(2)}`, role: 'GUEST', password: 'guest123' }),
};
```

**Quiz:** Why is a hand-rolled global "singleton page" module an antipattern in a Playwright framework, even though the Singleton pattern itself is legitimate elsewhere?
**Challenge:** Write a `TestDataBuilder` for a "product" entity with at least 4 fields (name, price, category, inStock), sensible defaults for all of them, and a fluent API to override any subset.

---

## Module 4.6 — Behavioral Patterns

**Core mechanics:**
- **Strategy Pattern**: swap an algorithm/behavior at runtime via a common interface — e.g., a pluggable `AuthStrategy` (UI login vs. API-token login vs. SSO) selected per environment/project without changing test code.
- **Command Pattern**: encapsulate an action (with its parameters) as an object that can be queued, logged, retried, or undone — useful for building custom retry wrappers around flaky third-party widget interactions.
- **Observer Pattern**: objects (custom reporters, listeners) subscribe to events (test start/end/fail) without the test runner needing to know about them explicitly — this is exactly how Playwright's own custom Reporter API works (Module 5.2).

**Best practices:** Model environment-specific or role-specific auth as a Strategy so the *same* test suite can run against dev (API-token login) and staging (full SSO flow) by swapping one implementation, not branching test code with `if (env === ...)`.
**Pitfalls:** Sprinkling `if (process.env.ENV === 'staging') { ... } else { ... }` branches throughout test/framework code instead of isolating environment-specific behavior behind a single Strategy interface — this scatters environment coupling everywhere instead of centralizing it.

**Code:**
```typescript
interface AuthStrategy { login(page: Page): Promise<void>; }

class ApiTokenAuth implements AuthStrategy {
  async login(page: Page) {
    const token = await getApiToken();
    await page.context().addCookies([{ name: 'session', value: token, url: baseURL }]);
  }
}
class UiLoginAuth implements AuthStrategy {
  async login(page: Page) {
    await page.goto('/login');
    await page.getByLabel('Username').fill('user');
    await page.getByRole('button', { name: 'Log in' }).click();
  }
}

const authStrategy: AuthStrategy = process.env.ENV === 'staging' ? new UiLoginAuth() : new ApiTokenAuth();
await authStrategy.login(page);
```

**Quiz:** Which existing Playwright feature already implements the Observer pattern natively, and what does that let third-party tools (like Allure) plug into?
**Challenge:** Design (interface + two implementations) a `NotificationStrategy` for sending test-failure alerts — one implementation posts to Slack, another writes to a local log file — selectable via config without changing any test code.

---

## Module 4.7 — Structural Patterns

**Core mechanics:**
- **Facade Pattern**: hide a multi-step, multi-page workflow behind one simple method call — e.g., `checkoutFlow.completeAsGuest(cart)` internally drives 4 pages' worth of interactions, but the test only sees one line.
- **Adapter Pattern**: wrap an inconsistent third-party API/library (e.g., a payment sandbox SDK, a visual-diff tool) behind a consistent internal interface, so if you swap vendors later, only the adapter changes — no test rewrites.
- **Decorator Pattern**: wrap a Locator/Page Object method to transparently add cross-cutting behavior (logging every action, auto-retry on a known-flaky third-party widget) without modifying the original method's code.

**Best practices:** Use a Facade for any workflow a test would otherwise need 15+ lines of low-level steps to express — the test should read as business intent (Module 4.3's Screenplay Task achieves a similar goal via a different vocabulary).
**Pitfalls:** Wrapping *every* Playwright action in custom Decorators "for consistency" — adds indirection and debugging friction for cross-cutting concerns you don't actually need broadly (e.g., logging every single click when only a handful of genuinely flaky third-party widgets need special handling).

**Code:**
```typescript
// Facade — one call hides a 4-page workflow
class CheckoutFacade {
  constructor(private page: Page) {}
  async completeAsGuest(item: string) {
    await this.page.goto('/cart');
    await this.page.getByRole('button', { name: `Add ${item}` }).click();
    await this.page.getByRole('link', { name: 'Checkout' }).click();
    await this.page.getByRole('button', { name: 'Continue as guest' }).click();
    await this.page.getByLabel('Email').fill('guest@example.com');
    await this.page.getByRole('button', { name: 'Place order' }).click();
  }
}

// Decorator — adds retry behavior to one specific flaky widget without touching Playwright internals
async function clickWithRetry(locator: Locator, attempts = 3) {
  for (let i = 0; i < attempts; i++) {
    try { return await locator.click({ timeout: 3000 }); }
    catch (e) { if (i === attempts - 1) throw e; }
  }
}
```

**Quiz:** What's the risk of applying the Decorator pattern (e.g., auto-retry wrappers) too broadly across a framework?
**Challenge:** Write an Adapter class wrapping a hypothetical third-party visual-diff SDK with an inconsistent API (`sdk.compare(imgA, imgB, opts)`) behind a clean internal interface `VisualComparator.diff(baseline, current): DiffResult` that your framework's tests actually use.

---

## Module 4.8 — Layered Framework Architecture

**Analogy:** A well-run kitchen separates the dining room (test layer, business intent) from the line cooks (business/workflow layer) from the pantry and equipment (UI interaction / driver layer) — a menu change shouldn't require re-wiring the stoves.

**Core mechanics — the 4 layers:**
1. **Test Layer**: `*.spec.ts` files — pure business intent, calls into layer 2, contains assertions only.
2. **Business/Workflow Layer**: Facades/Screenplay Tasks — orchestrates multi-step flows spanning multiple Page Objects/components.
3. **UI Interaction Layer**: Page Objects/Components — encapsulates locators and raw Playwright actions for one screen/component.
4. **Driver/Config Layer**: fixtures, `playwright.config.ts`, environment resolution, browser/context lifecycle — the plumbing every layer above depends on but never sees directly.

Data flows top-down (test → workflow → UI interaction → driver); failures surface bottom-up (a driver-layer timeout bubbles up as a failed assertion at the test layer, but the *trace* shows exactly which layer boundary it happened at).

**Best practices:** Enforce the boundary strictly — a `*.spec.ts` file should never contain a raw CSS selector; if it does, that's a layering violation to fix by moving it into layer 3.
**Pitfalls:** Letting business logic leak into the UI interaction layer (e.g., a Page Object method that decides *which* discount code to apply based on business rules) — couples layers that should be independently replaceable.

**Quiz:** If a test file contains a raw `page.locator('.btn-primary')` call, which architectural layer boundary has been violated, and how would you fix it?
**Challenge:** Take the `CheckoutFacade` from Module 4.7 and explicitly map its pieces onto the 4 layers — identify which lines belong in the test file vs. the workflow layer vs. the Page Object layer.

---

## Module 4.9 — Data-Driven & Hybrid Framework Design

**Core mechanics:**
- **Data-driven testing**: the same test logic runs against externalized data sets (JSON/YAML/CSV/DB) via `for (const case of testCases) { test(case.name, ...) }` — separates "what varies" from "what's verified."
- **Keyword-driven testing**: test steps are expressed as data (a spreadsheet/table of keywords like `LOGIN`, `CLICK`, `ASSERT_TEXT`) interpreted by a generic engine — maximizes non-coder authorability, at the cost of debuggability and type safety; largely superseded in modern Playwright frameworks by code-based approaches, but still relevant in some enterprise/regulated contexts.
- **BDD hybrid** (Cucumber + Playwright, via `playwright-bdd` or similar): Gherkin `.feature` files (`Given/When/Then`) map to step definitions that call into your Page Object/Facade layer — trades some indirection for stakeholder-readable specs.

**Best practices:** Use plain data-driven `test.each`-style loops for the vast majority of parameterized cases — reach for full BDD/Gherkin only when a real stakeholder (product/QA lead) actually reads and reviews the feature files; otherwise it's ceremony without payoff.
**Pitfalls:** Adopting Cucumber/Gherkin because it "looks more professional" when no non-technical stakeholder actually consumes the `.feature` files — adds a translation layer (steps → code) that pure TypeScript wouldn't need, for zero real benefit.

**Code:**
```typescript
// Data-driven — plain and effective for most cases
const invalidLogins = [
  { username: '', password: 'x', expectedError: 'Username is required' },
  { username: 'a', password: '', expectedError: 'Password is required' },
];
for (const { username, password, expectedError } of invalidLogins) {
  test(`shows error for username="${username}" password="${password}"`, async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Username').fill(username);
    await page.getByLabel('Password').fill(password);
    await page.getByRole('button', { name: 'Log in' }).click();
    await expect(page.getByText(expectedError)).toBeVisible();
  });
}
```

**Quiz:** Under what specific circumstance does BDD/Gherkin's added indirection actually pay for itself in a Playwright framework?
**Challenge:** Convert 3 hardcoded near-duplicate login-validation tests into a single data-driven loop, externalizing the test cases into a separate JSON file imported at the top of the spec.

---

## Module 4.10 — Configuration & Environment Management Patterns

**Core mechanics:**
- Combine **Factory** (produce the right config object) with **Strategy** (the right *behavior*, e.g. auth method) per environment — a single `getEnvironmentConfig(env)` factory returns `{ baseURL, authStrategy, apiUrl, featureFlags }` for `dev`/`staging`/`prod`-like targets, consumed uniformly by the rest of the framework.
- Environment resolution should happen **once**, at config/global-setup time — not scattered as `process.env.X` reads throughout individual test files.
- Secrets (API tokens, test-account passwords) belong in CI secret stores / `.env` files excluded from version control — never hardcoded in spec files, even for "throwaway" test accounts (they leak into git history permanently).

**Best practices:** One `environments/` module exporting a typed config object per environment, selected via a single `ENV` variable — everything else in the framework imports from that resolved object, never reads `process.env` directly outside it.
**Pitfalls:** `process.env.SOME_FLAG === 'true'` checks scattered across dozens of files — impossible to audit what environment-dependent behavior exists without grepping the whole codebase.

**Code:**
```typescript
// environments/index.ts
type EnvConfig = { baseURL: string; apiURL: string; authStrategy: AuthStrategy };
const configs: Record<string, EnvConfig> = {
  dev: { baseURL: 'http://localhost:3000', apiURL: 'http://localhost:4000', authStrategy: new ApiTokenAuth() },
  staging: { baseURL: 'https://staging.example.com', apiURL: 'https://api-staging.example.com', authStrategy: new UiLoginAuth() },
};
export const env = configs[process.env.ENV ?? 'dev'];
```

**Quiz:** Why is centralizing environment resolution into a single module preferable to scattering `process.env` checks throughout the codebase?
**Challenge:** Design an `environments/index.ts` module supporting `dev`/`staging`/`prod`-like configs, each with a different `AuthStrategy` (reusing Module 4.6), consumed by a single `env` export used everywhere else.

---

## Module 4.11 — Capstone: Design a Full Framework Skeleton

**The exercise:** Using everything from Modules 4.1–4.10, design (folder structure + key file skeletons, not a full implementation) a Playwright framework for a mid-sized e-commerce app with: guest + logged-in user flows, an admin portal, checkout involving a third-party payment redirect, and a need to run against 3 environments in CI.

**Deliverable checklist to defend in review:**
- [ ] Folder structure showing the 4 layers (Module 4.8) clearly separated.
- [ ] At least one composed Page Object using components, not deep inheritance (4.2).
- [ ] A fixture graph with at least one worker-scoped and one test-scoped fixture with a real dependency between them (4.4).
- [ ] A Facade or Task for the checkout workflow spanning the payment popup (ties back to 3.4 multi-context).
- [ ] An `AuthStrategy` selected per environment (4.6, 4.10).
- [ ] A Builder or Factory for test data (4.5).
- [ ] A justification for *not* using Screenplay/BDD if you chose plain POM+fixtures instead — or a justification for using them if you did (4.3/4.9) — design decisions should be defensible, not default.

**This is the deliverable to bring to our live session** — I'll review it the way I'd review a senior engineer's framework RFC: pushing on every "why," not just "does it work."
