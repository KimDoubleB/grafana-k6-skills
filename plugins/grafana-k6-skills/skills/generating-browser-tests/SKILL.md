---
name: generating-browser-tests
description: Use when creating k6 browser-based performance tests, testing frontend performance, measuring Web Vitals (LCP, FCP, CLS, INP, TTFB), or automating browser interactions for load testing. Use when the user mentions browser testing, page load testing, E2E performance, or Chromium-based testing with k6.
---

# Generating k6 Browser Tests

Generate browser-based E2E performance tests using the k6/browser module. Tests real browser interactions with Chromium, measures Web Vitals, and supports hybrid protocol+browser testing.

## Quick Start

```javascript
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
    await page.goto('https://example.com');
    await page.waitForLoadState('networkidle');

    const title = await page.title();
    check(null, { 'page title loaded': () => title !== '' });

    await page.screenshot({ path: 'screenshot.png' });
  } finally {
    await page.close(); // Always close pages
  }
}
```

## Key Concepts

- **All browser functions are `async`** — use `await` and `async function`
- **Always close pages** — use `try/finally` to ensure `page.close()` is called
- **Browser type** — only `'chromium'` is supported
- **Browser scenario** — requires `options.browser.type` in scenario config
- **Avoid `networkidle`** — `waitForLoadState('networkidle')` may never fire on chatty pages; prefer `'load'` or `locator.waitFor()` when possible
- **Resource intensive** — browser VUs consume more resources; start with few VUs

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
  },
};
```

## Common Test Patterns

### Login Flow

```javascript
export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto('https://app.example.com/login');

    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await page.waitForNavigation();

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
    await page.waitForLoadState('networkidle');

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

    await page.locator('a[href="/products"]').click();
    await page.waitForNavigation();
    await page.screenshot({ path: 'products.png' });

    await page.locator('.product-card').first().click();
    await page.waitForLoadState('networkidle');
    await page.screenshot({ path: 'product-detail.png' });
  } finally {
    await page.close();
  }
}
```

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

## Advanced Features

For detailed API reference including BrowserContext, Keyboard, Mouse, Touchscreen, and advanced patterns:

See [reference/browser-api.md](reference/browser-api.md) — Complete Page, Locator, BrowserContext API with all methods and parameters

See [reference/web-vitals.md](reference/web-vitals.md) — Web Vitals metrics explanation, threshold guidance, and custom performance measurement

## Related Skills

- For protocol-level load tests (HTTP, gRPC, WebSocket): `/k6:generating-api-load-tests`
- For scenario and executor configuration: `/k6:designing-test-scenarios`
- For interpreting test results: `/k6:analyzing-test-results`
