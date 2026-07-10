# Phase 5: Architecture & Ecosystem — Production-Grade Systems

---

## Module 5.1 — CI/CD Integration Patterns

**Core mechanics:**
- Playwright's official Docker images (`mcr.microsoft.com/playwright:v<version>-<os>`) bundle matching browser binaries pre-installed — avoids the classic "works locally, browser download times out in CI" failure mode; **pin the image tag to your installed `@playwright/test` version** or binary/library mismatches cause cryptic launch failures.
- GitHub Actions caching: cache `~/.cache/ms-playwright` keyed on the Playwright version to skip re-downloading browsers every run.
- Sharded matrix builds (`strategy.matrix.shard: [1,2,3,4]` + `--shard=${{ matrix.shard }}/4`) parallelize wall-clock time across CI runners; must merge blob reports afterward (Module 5.2).

**Best practices:** Pin browser + framework versions together explicitly; run smoke tests on every PR, full cross-browser+shard matrix on merge-to-main/nightly.
**Pitfalls:** Upgrading `@playwright/test` without re-running `npx playwright install` in the CI image/cache — version skew between the library and installed browser binaries causes obscure protocol-mismatch errors.

**Code:**
```yaml
# .github/workflows/playwright.yml (excerpt)
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test --shard=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        with: { name: blob-report-${{ matrix.shard }}, path: blob-report/ }
```

**Quiz:** Why must the Playwright Docker image tag (or `playwright install` step) stay in lockstep with your `@playwright/test` npm version?
**Challenge:** Write a GitHub Actions matrix config that shards across 3 runners AND fans out across 2 browser projects (chromium/webkit) — 6 total parallel jobs — and uploads each shard's report artifact with a unique name.

---

## Module 5.2 — Reporting Ecosystems

**Core mechanics:**
- The built-in **HTML reporter** bundles traces, screenshots, and videos into a single browsable report — `npx playwright show-report`.
- **Blob reporter** (`--reporter=blob`) produces a merge-ready intermediate format specifically for sharded CI runs — after all shards finish, `npx playwright merge-reports` combines them into one unified HTML report.
- **Allure** integration (`allure-playwright`) plugs in via the Observer-pattern reporter API (Module 4.6) — useful when a broader org already standardizes reporting/dashboards across multiple test frameworks, not just Playwright.
- Custom reporters implement a simple interface (`onBegin`, `onTestEnd`, `onEnd`) — e.g., posting a Slack summary automatically at suite completion.

**Best practices:** Use `blob` reporter + `merge-reports` for sharded CI so you get one report, not N disconnected ones. Reserve third-party reporters (Allure, etc.) for genuine cross-team reporting standardization needs.
**Pitfalls:** Sharding CI without a merge step — each shard's HTML report is disconnected, so nobody has a single view of full suite health, and failures get missed because they're scattered across 4 separate artifacts.

**Code:**
```typescript
// custom-reporter.ts — Observer pattern in action
import type { Reporter, TestCase, TestResult } from '@playwright/test/reporter';

class SlackReporter implements Reporter {
  private failures: string[] = [];
  onTestEnd(test: TestCase, result: TestResult) {
    if (result.status === 'failed') this.failures.push(test.title);
  }
  async onEnd() {
    if (this.failures.length) await postToSlack(`${this.failures.length} tests failed: ${this.failures.join(', ')}`);
  }
}
export default SlackReporter;
```

**Quiz:** Why does a sharded CI run specifically require the `blob` reporter + merge step, rather than just using the default HTML reporter on each shard?
**Challenge:** Write a minimal custom reporter that counts total pass/fail/flaky counts and prints a one-line summary to console on `onEnd()`.

---

## Module 5.3 — Test Data Management & Environment Strategy

**Core mechanics:**
- **Seeding**: create required preconditions via API calls (Module 2.8) in `beforeEach`/fixtures rather than relying on pre-existing "known" database state, which drifts and causes environment-specific failures.
- **Teardown**: always clean up test-created data (via fixture teardown, Module 2.1/4.4) so re-runs are idempotent and parallel workers don't collide on shared records.
- **Ephemeral environments**: spinning up a fresh, isolated environment (or DB schema/namespace) per PR/branch eliminates cross-team test-data collisions entirely — the gold standard for larger orgs, at added infra cost.
- **Secrets handling**: CI secret stores (GitHub Actions Secrets, Vault) injected as env vars at runtime — never committed, never logged in plaintext (mask sensitive values in custom reporters/logs).

**Best practices:** Every test that creates data must also delete it (or run against a scoped/ephemeral environment that's discarded wholesale) — "leave no trace" is the isolation contract that makes parallel execution safe.
**Pitfalls:** Relying on a shared staging DB's "known" seed record (e.g., "user ID 42 always exists") — another team's cleanup script or a schema migration silently breaks every test depending on that assumption.

**Quiz:** Why does relying on "known" pre-existing data in a shared staging environment become a liability as a test suite and organization scale?
**Challenge:** Design a fixture (using Module 4.4 patterns) that creates a test user via API in setup and guarantees its deletion in teardown even if the test itself throws an assertion error mid-run.

---

## Module 5.4 — Scaling to Hundreds of Specs

**Core mechanics:**
- **Monorepo structuring**: group specs by domain/feature (`tests/checkout/`, `tests/admin/`) rather than by test type — colocate related Page Objects/fixtures with the specs that use them, with genuinely shared infra in a top-level `framework/` or `support/` package.
- **Tagging** (`test('...', { tag: '@smoke' }, ...)`) lets you run subsets via `--grep @smoke` — critical for fast PR-gating suites vs. full nightly regression.
- **`testDependencies`** (project-level `dependencies`) let you sequence setup projects (Module 2.5's auth setup) before dependent projects run, without manual ordering logic.

**Best practices:** Maintain a fast `@smoke` tagged subset (2-5 minutes) that gates every PR; run the full suite on a schedule/merge-to-main. Structure folders by domain, not by "unit/integration/e2e" test-type buckets, so ownership maps cleanly to feature teams.
**Pitfalls:** One flat `tests/` folder with 500 files and no tagging strategy — every PR either runs everything (slow feedback) or nothing gets consistently run pre-merge (regressions slip through).

**Code:**
```typescript
test('guest checkout completes successfully', { tag: '@smoke' }, async ({ page }) => { /* ... */ });
```
```bash
npx playwright test --grep @smoke     # fast PR-gating subset
```

**Quiz:** What's the tagging strategy that lets a 500-spec suite still provide fast (<5 min) PR feedback without skipping regression coverage entirely?
**Challenge:** Propose a folder structure for a monorepo with 3 feature teams (checkout, admin, search) sharing one Playwright config and a common `support/` package — show where fixtures, Page Objects, and specs each live.

---

## Module 5.5 — Playwright + AI

**Core mechanics:**
- **Codegen** (Module 3.2) is the original "AI-adjacent" tool — records interactions, emits starting-point code; not AI-based itself, but the precursor UX these newer tools build on.
- **Self-healing locators**: emerging tools/services detect when a previously-passing locator no longer matches and suggest (or auto-apply) an updated locator based on semantic similarity to the original element — reduces maintenance burden from routine markup churn, but requires human review to avoid silently "healing" onto the *wrong* element after a genuine behavior change.
- **MCP (Model Context Protocol)-based Playwright tooling**: exposes Playwright's browser-control primitives as tools an AI agent can call directly — enabling an LLM agent to drive a real browser (navigate, click, extract content) as part of a larger agentic workflow, distinct from using AI merely to *author* test code beforehand.
- Per Microsoft's "Complete Playwright E2E Story" positioning: AI is being layered in at multiple points — authoring (codegen-plus-LLM test generation), maintenance (self-healing), and execution (MCP-driven agents) — not as a replacement for the deterministic test-runner core, which remains unchanged.

**Best practices:** Use AI-assisted authoring as a fast first draft, always followed by human review against the locator-priority guidance from Module 1.3 — AI-generated locators tend to default to brittle CSS/XPath unless explicitly guided otherwise.
**Pitfalls:** Trusting auto-"healed" locators without review — a locator that silently repoints to a visually-similar-but-semantically-different element (e.g., a different row in a table) turns a would-be loud test failure into a silent false negative.

**Quiz:** What's the specific risk of unreviewed self-healing locators, beyond just "maintenance convenience"?
**Challenge:** Take an AI/codegen-produced test snippet (hypothetically full of brittle nth-child CSS selectors) and manually refactor it to use role/label-based locators per Module 1.3's priority guidance.

---

## Module 5.6 — Extending Playwright

**Core mechanics:**
- **Custom fixtures/plugins** (Module 4.4) are Playwright's primary extension point — there's no separate "plugin system" beyond the fixture and reporter APIs; this is intentional, keeping one consistent extension mechanism.
- **Custom matchers**: `expect.extend({...})` adds domain-specific assertions (e.g., `expect(price).toBeApproximatelyEqual(19.99, 0.01)`) that read naturally in tests and centralize comparison logic used repeatedly.
- Custom reporters (Module 5.2) and custom fixtures together cover the vast majority of "framework extension" needs without ever touching Playwright's internals.

**Best practices:** Before reaching for a workaround/hack, check whether a custom fixture, custom matcher, or custom reporter already covers the need — Playwright's extension surface is intentionally comprehensive.
**Pitfalls:** Monkey-patching Playwright's internal classes/prototypes to add behavior — unsupported, breaks on version upgrades, and bypasses the well-defined fixture/matcher extension points that exist for exactly this purpose.

**Code:**
```typescript
// Custom matcher
expect.extend({
  toBeApproximatelyEqual(received: number, expected: number, tolerance = 0.01) {
    const pass = Math.abs(received - expected) <= tolerance;
    return { pass, message: () => `expected ${received} to be within ${tolerance} of ${expected}` };
  },
});
```

**Quiz:** Why is monkey-patching Playwright internals discouraged when custom fixtures/matchers/reporters already exist as sanctioned extension points?
**Challenge:** Write a custom matcher `toHaveValidEmailFormat()` usable as `expect(emailString).toHaveValidEmailFormat()`, including a clear failure message.

---

## Module 5.7 — Performance Testing Hooks via CDP

**Core mechanics:**
- Playwright exposes a raw **CDP session** (`context.newCDPSession(page)`, Chromium-only) — gives access to protocol domains beyond Playwright's own high-level API: `Performance.getMetrics`, `Tracing.start/stop` (full Chrome performance trace export), and code coverage (`page.coverage.startJSCoverage()`).
- This is explicitly a **Chromium-only escape hatch** — Firefox/WebKit don't expose CDP, so performance-hook code is inherently non-portable across the `projects` matrix (Module 3.7) and should be isolated to a Chromium-specific project/suite.
- Useful for lightweight "performance smoke tests" (e.g., asserting a page's JS heap size or paint timing doesn't regress past a threshold) — not a replacement for dedicated load-testing tools (k6, Lighthouse CI) which test server-side scalability under concurrent load, a different axis entirely.

**Best practices:** Treat CDP-based performance assertions as regression guardrails on a few critical pages, not a comprehensive performance-testing strategy — pair with Lighthouse CI or similar for real performance auditing.
**Pitfalls:** Writing CDP-dependent helper code without guarding it behind a Chromium-only check — it silently breaks (or must be skipped) when the same spec runs under the `webkit`/`firefox` projects.

**Code:**
```typescript
test('homepage JS heap stays within budget', async ({ page, browserName }) => {
  test.skip(browserName !== 'chromium', 'CDP metrics are Chromium-only');
  const client = await page.context().newCDPSession(page);
  await page.goto('/');
  const { metrics } = await client.send('Performance.getMetrics');
  const heap = metrics.find((m) => m.name === 'JSHeapUsedSize')?.value ?? 0;
  expect(heap).toBeLessThan(50 * 1024 * 1024); // 50MB budget
});
```

**Quiz:** Why must CDP-based performance checks be explicitly guarded or isolated within a multi-browser `projects` matrix?
**Challenge:** Write a Chromium-only test that starts JS coverage (`page.coverage.startJSCoverage()`), navigates a page, stops coverage, and asserts that overall JS usage percentage exceeds some minimum threshold (a rough "dead code" signal).

---

## Module 5.8 — Final Capstone: Full System Design Review

**The exercise:** Bringing together every phase, produce (and defend live) a complete production E2E framework design for a real or realistic app of your choosing, covering:

1. **Architecture** (Phase 4): layered structure, chosen pattern set (POM vs. Screenplay, justified), fixture graph.
2. **CI/CD** (5.1): pipeline design, sharding/matrix strategy, browser coverage.
3. **Reporting** (5.2): report format, merge strategy, any custom reporter needs.
4. **Data/Env strategy** (5.3, 4.10): seeding/teardown approach, environment config strategy, secrets handling.
5. **Scale strategy** (5.4): folder structure, tagging scheme, fast-feedback vs. full-regression split.
6. **Extensibility & AI tooling stance** (5.5, 5.6): where (if anywhere) AI-assisted authoring fits your team's workflow, and what human-review gate you'd put around it.
7. **Explicit non-goals**: what you deliberately did *not* build (e.g., "no custom Screenplay layer — team size doesn't justify it," "no CDP performance hooks — covered by a separate Lighthouse CI pipeline instead").

This is a system-design interview in miniature — the goal isn't a working repo, it's a defensible set of trade-off decisions. Bring it to our live session and I'll review it the way a staff engineer would review an RFC: probing every "why," and specifically pushing on your non-goals list to make sure it's a deliberate choice, not an oversight.
