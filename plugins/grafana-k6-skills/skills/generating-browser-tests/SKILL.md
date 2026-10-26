---
name: generating-browser-tests
description: Use when creating k6 browser-based performance tests, testing frontend performance, measuring Web Vitals (LCP, FCP, CLS, INP, TTFB), or automating browser interactions for load testing. Use when the user mentions browser testing, page load testing, E2E performance, or Chromium-based testing with k6.
---

# Generating k6 Browser Tests

Generate browser-based E2E performance tests using the k6/browser module. Tests real browser interactions with Chromium, measures Web Vitals, and supports hybrid protocol+browser testing.

## k6 Version Compatibility

Written for **k6 v2.3.0** (current stable). The v1.8.x maintenance line is supported where noted; differences are marked `v1.x:` inline. Before generating, determine the version of the environment that will run the test:

1. Run `k6 version` where the test will execute (prints `k6 v2.3.0 (...)`). For Docker use a `*-with-browser` tag (`grafana/k6:2.3.0-with-browser`); for CI read the `setup-k6-action` `k6-version` input (with `browser: true`); for k6-operator read `spec.runner.image`.
2. If no version is visible, ask the user. Otherwise assume v2.x and record the assumption in a header comment of the generated script, for example `// Target: k6 v2.3.0`.
3. Apply the notes below when the target is v1.x.

**Version notes for this skill:**
- `chromium.connectOverCDP(wsEndpoint)` (attach to a running browser) needs v2.2.0+. `v1.x:` set the `K6_BROWSER_WS_URL` environment variable instead.
- `browser.newContext({ proxy })` and `locator.isInViewport()` need v2.1.0+. `v1.x:` use `K6_BROWSER_ARGS='proxy-server=host:port'`.
- `browser_web_vital_fid` was removed in v2.0.0 (web-vitals v5). `v1.x:` still emitted; never put a threshold on it, use `browser_web_vital_inp`.
- Run a browser script once with `k6 run --once script.js` (v2.3.0+); it keeps the scenario's `options.browser`. `v1.x:` `--vus 1 --iterations 1` discards the browser options and the test fails to launch Chromium, so keep the scenario and set `iterations: 1` in it.
- Since v2.3.0, Web Vitals from intermediate pages in the same tab are kept, so flow-level averages and percentiles differ from v1.x/v2.2 runs. Threshold per page with the `url` tag.
- Locator actions retry automatically since v1.8.0 / v2.0.0.

For the full feature/version matrix see `/k6:operating-k6-in-ci-cd` (reference/version-compatibility.md).

## Quick Start

```javascript
// Target: k6 v2.3.0
import { browser } from 'k6/browser';
import { check } from 'k6';

export const options = {
  scenarios: {
    ui: {
      executor: 'shared-iterations',
      iterations: 5,
      vus: 1,
      options: {
        browser: {
          type: 'chromium',
        },
      },
    },
  },
  thresholds: {
    browser_web_vital_lcp: ['p(90)<2500'],
    browser_web_vital_fcp: ['p(90)<1800'],
    browser_web_vital_cls: ['p(95)<0.1'],
  },
};

export default async function () {
  const page = await browser.newPage();

  try {
    await page.goto('https://quickpizza.grafana.com/'); // waits for the load event

    const title = await page.title();
    check(null, { 'page title loaded': () => title !== '' });

    await page.screenshot({ path: 'screenshot.png' });
  } finally {
    await page.close(); // Always close pages so the last Web Vitals are flushed
  }
}
```

Run it once to check the script before adding load: `k6 run --once script.js` (v2.3.0+).

## Key Concepts

- **All browser functions are `async`** — use `await` and `async function`
- **Always close pages** — use `try/finally` to ensure `page.close()` is called; an abrupt exit can drop Web Vitals that were not yet emitted
- **Browser type** — only `'chromium'` is supported
- **Browser scenario** — requires `options.browser.type` in scenario config (not needed for `chromium.connectOverCDP()`)
- **Click + navigation** — start waiting before the click: `await Promise.all([page.waitForNavigation(), locator.click()])`. Awaiting `click()` first can miss a fast navigation
- **Avoid `networkidle`** — `waitForLoadState('networkidle')` may never fire on chatty pages; prefer `'load'` or `locator.waitFor()` when possible
- **Resource intensive** — browser VUs consume more resources; start with few VUs
- **Chromium flags** — pass `K6_BROWSER_ARGS` without the `--` prefix, for example `K6_BROWSER_ARGS='ignore-certificate-errors,proxy-server=127.0.0.1:8080'`; `no-sandbox` is needed when running as root in Docker, CI, or Kubernetes

## Element Selection

### Locator API (Recommended)

```javascript
// CSS selector
const btn = page.locator('button.submit');

// Semantic selectors (preferred for resilience)
const submitBtn = page.getByRole('button', { name: 'Submit' });
const email = page.getByLabel('Email');
const search = page.getByPlaceholder('Search...');
const heading = page.getByText('Welcome');
const card = page.getByTestId('user-card');
```

### Common Interactions

```javascript
// Click
await page.locator('button').click();

// Fill text input
await page.getByLabel('Username').fill('testuser');

// Select dropdown option
await page.locator('select#country').selectOption('US');

// Checkbox
await page.getByRole('checkbox', { name: 'Terms' }).check();

// Type character by character (with key events)
await page.locator('#search').type('search query', { delay: 50 });

// Press key
await page.locator('#search').press('Enter');

// Hover
await page.locator('.menu-item').hover();
```

### Query Element State

```javascript
const isVisible = await page.locator('.modal').isVisible();
const isEnabled = await page.locator('button').isEnabled();
const text = await page.locator('.message').textContent();
const value = await page.locator('input').inputValue();
const count = await page.locator('li.item').count();
```

## Web Vitals

k6/browser automatically collects Core Web Vitals:

| Metric | Name | Good Threshold |
|--------|------|----------------|
| `browser_web_vital_lcp` | Largest Contentful Paint | < 2500ms |
| `browser_web_vital_fcp` | First Contentful Paint | < 1800ms |
| `browser_web_vital_cls` | Cumulative Layout Shift | < 0.1 |
| `browser_web_vital_inp` | Interaction to Next Paint | < 200ms |
| `browser_web_vital_ttfb` | Time to First Byte | < 600ms |

```javascript
export const options = {
  thresholds: {
    'browser_web_vital_lcp': ['p(90)<2500'],
    'browser_web_vital_fcp': ['p(90)<1800'],
    'browser_web_vital_cls': ['p(95)<0.1'],
    'browser_web_vital_inp': ['p(90)<200'],
    // Per-page thresholds via the url tag
    'browser_web_vital_lcp{url:https://quickpizza.grafana.com/}': ['p(90)<1500'],
  },
};
```

Since k6 v2.3.0 every page a flow navigates through in the same tab contributes its own Web Vitals samples (earlier versions could lose the intermediate pages). Flow-level aggregates therefore shift after upgrading; prefer `url`-tagged thresholds for per-page targets. `browser_web_vital_fid` no longer exists (removed in v2.0.0).

## Common Test Patterns

### Login Flow

```javascript
export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto('https://app.example.com/login');

    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('password123');

    await Promise.all([
      page.waitForNavigation(),
      page.getByRole('button', { name: 'Sign in' }).click(),
    ]);

    check(null, {
      'redirected to dashboard': () => page.url().includes('/dashboard'),
    });
  } finally {
    await page.close();
  }
}
```

### Form Submission

```javascript
export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto('https://app.example.com/contact');

    await page.getByLabel('Name').fill('Load Test User');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Message').fill('Performance test submission');
    await page.locator('select#department').selectOption('support');
    await page.getByRole('checkbox', { name: 'Subscribe' }).check();

    await page.getByRole('button', { name: 'Submit' }).click();
    await page.locator('.success-message').waitFor({ state: 'visible' }); // wait for the result, not for networkidle

    const confirmation = await page.locator('.success-message').textContent();
    check(null, {
      'form submitted': () => confirmation.includes('Thank you'),
    });
  } finally {
    await page.close();
  }
}
```

### Page Navigation with Screenshots

```javascript
export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto('https://app.example.com');
    await page.screenshot({ path: 'home.png', fullPage: true });

    await Promise.all([page.waitForNavigation(), page.locator('a[href="/products"]').click()]);
    await page.screenshot({ path: 'products.png' });

    await Promise.all([page.waitForNavigation(), page.locator('.product-card').first().click()]);
    await page.screenshot({ path: 'product-detail.png' });
  } finally {
    await page.close();
  }
}
```

### Connect to an Existing Browser (`connectOverCDP`, v2.2.0+)

Attach to a Chromium that is already running (local `--remote-debugging-port`, a browser grid, or a cloud browser provider) instead of letting k6 launch one. No `options.browser` scenario block is needed, and the endpoint is a runtime value, so it can come from `setup()`.

```javascript
// Target: k6 v2.3.0
import { chromium } from 'k6/browser';

export default async function () {
  const browser = await chromium.connectOverCDP(__ENV.CDP_WS_URL); // ws://host:9222/devtools/browser/<id>
  const page = await browser.newPage();
  try {
    await page.goto('https://quickpizza.grafana.com/');
  } finally {
    await page.close();
    await browser.close(); // releases the connection only; the browser keeps running
  }
}
```

Read the endpoint from `http://localhost:9222/json/version` (`webSocketDebuggerUrl`). The returned browser is the only one that exposes `close()`; k6 closes the connection at the end of the iteration anyway. `v1.x:` use `K6_BROWSER_WS_URL=ws://...` with a normal `browser` scenario.

## Hybrid Testing (Browser + Protocol)

Combine browser tests with protocol-level load tests:

```javascript
import { browser } from 'k6/browser';
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    // High-volume API load
    api_load: {
      exec: 'apiTest',
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
    },
    // Low-volume browser tests (resource intensive)
    browser_test: {
      exec: 'browserTest',
      executor: 'constant-vus',
      vus: 2,
      duration: '5m',
      options: { browser: { type: 'chromium' } },
    },
  },
};

export function apiTest() {
  const res = http.get('https://api.example.com/products');
  check(res, { 'API 200': (r) => r.status === 200 });
}

export async function browserTest() {
  const page = await browser.newPage();
  try {
    await page.goto('https://app.example.com');
    // Browser interactions...
  } finally {
    await page.close();
  }
}
```

While developing one half, run only that scenario (v2.3.0+): `k6 run --scenario browser_test test.js` keeps the browser options, `k6 run --scenario api_load test.js` keeps the arrival rate. Add `--once` to run the selected scenario a single time.

## Advanced Features

For detailed API reference including BrowserContext, Keyboard, Mouse, Touchscreen, and advanced patterns:

See [reference/browser-api.md](reference/browser-api.md) — Complete Page, Locator, BrowserContext API with all methods and parameters

See [reference/web-vitals.md](reference/web-vitals.md) — Web Vitals metrics explanation, threshold guidance, and custom performance measurement

## Related Skills

- For protocol-level load tests (HTTP, gRPC, WebSocket): `/k6:generating-api-load-tests`
- For scenario and executor configuration: `/k6:designing-test-scenarios`
- For interpreting test results: `/k6:analyzing-test-results`
- For the k6 version matrix and browser images in CI/Kubernetes: `/k6:operating-k6-in-ci-cd`
