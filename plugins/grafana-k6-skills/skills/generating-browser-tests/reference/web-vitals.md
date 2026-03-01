# k6 Browser Web Vitals Reference

## Core Web Vitals

| Metric | Full Name | What It Measures | Good | Needs Improvement | Poor |
|--------|-----------|------------------|------|-------------------|------|
| LCP | Largest Contentful Paint | Largest visible content render time | < 2.5s | 2.5s - 4s | > 4s |
| FCP | First Contentful Paint | First content render time | < 1.8s | 1.8s - 3s | > 3s |
| CLS | Cumulative Layout Shift | Visual stability score | < 0.1 | 0.1 - 0.25 | > 0.25 |
| INP | Interaction to Next Paint | Input responsiveness | < 200ms | 200ms - 500ms | > 500ms |
| TTFB | Time to First Byte | Server response time | < 600ms | 600ms - 1.5s | > 1.5s |

## k6 Metric Names

| k6 Metric | Type | Description |
|-----------|------|-------------|
| `browser_web_vital_lcp` | Trend | Largest Contentful Paint timing |
| `browser_web_vital_fcp` | Trend | First Contentful Paint timing |
| `browser_web_vital_cls` | Trend | Cumulative Layout Shift score |
| `browser_web_vital_inp` | Trend | Interaction to Next Paint timing |
| `browser_web_vital_ttfb` | Trend | Time to First Byte timing |
| `browser_web_vital_fid` | Trend | First Input Delay (deprecated, use INP) |

## Additional Browser Metrics

| k6 Metric | Type | Description |
|-----------|------|-------------|
| `browser_data_received` | Counter | Data downloaded (bytes) |
| `browser_data_sent` | Counter | Data uploaded (bytes) |
| `browser_http_req_duration` | Trend | Browser HTTP request duration |
| `browser_http_req_failed` | Rate | Browser failed request rate |

## Threshold Configuration

### Recommended Thresholds

```javascript
export const options = {
  thresholds: {
    // Core Web Vitals (Good targets)
    'browser_web_vital_lcp': ['p(90)<2500'],   // 90% of pages LCP < 2.5s
    'browser_web_vital_fcp': ['p(90)<1800'],   // 90% of pages FCP < 1.8s
    'browser_web_vital_cls': ['p(95)<0.1'],    // 95% of pages CLS < 0.1
    'browser_web_vital_inp': ['p(90)<200'],    // 90% of interactions INP < 200ms
    'browser_web_vital_ttfb': ['p(90)<600'],   // 90% of pages TTFB < 600ms

    // Browser network metrics
    'browser_http_req_failed': ['rate<0.01'],  // Browser request error rate < 1%
  },
};
```

### Strict Thresholds (High-Performance)

```javascript
thresholds: {
  'browser_web_vital_lcp': ['p(90)<1500', 'p(99)<2500'],
  'browser_web_vital_fcp': ['p(90)<1000', 'p(99)<1800'],
  'browser_web_vital_cls': ['p(99)<0.05'],
  'browser_web_vital_inp': ['p(90)<100', 'p(99)<200'],
  'browser_web_vital_ttfb': ['p(90)<300', 'p(99)<600'],
}
```

## Custom Performance Measurement

### Using Performance API

```javascript
export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto('https://app.example.com');

    // Mark start of user action
    await page.evaluate(() => window.performance.mark('action-start'));

    // Perform action
    await page.getByRole('button', { name: 'Load Data' }).click();
    await page.waitForLoadState('networkidle');

    // Mark end and measure
    await page.evaluate(() => {
      window.performance.mark('action-end');
      window.performance.measure('action-time', 'action-start', 'action-end');
    });

    const duration = await page.evaluate(() =>
      window.performance.getEntriesByName('action-time')[0].duration
    );

    console.log(`Action took ${duration}ms`);
  } finally {
    await page.close();
  }
}
```

### Using k6 Custom Metrics

```javascript
import { browser } from 'k6/browser';
import { Trend } from 'k6/metrics';

const pageLoadTime = new Trend('page_load_time');
const loginDuration = new Trend('login_duration');

export default async function () {
  const page = await browser.newPage();
  try {
    const start = Date.now();
    await page.goto('https://app.example.com');
    await page.waitForLoadState('networkidle');
    pageLoadTime.add(Date.now() - start);

    const loginStart = Date.now();
    await page.getByLabel('Email').fill('user@test.com');
    await page.getByLabel('Password').fill('pass123');
    await page.getByRole('button', { name: 'Login' }).click();
    await page.waitForNavigation();
    loginDuration.add(Date.now() - loginStart);
  } finally {
    await page.close();
  }
}
```

## Interpreting Results

### LCP (Largest Contentful Paint)
- **What affects it:** Large images, web fonts, CSS background images, block-level text
- **Optimization:** Optimize images, preload critical resources, use CDN, server-side render

### FCP (First Contentful Paint)
- **What affects it:** Server response time, render-blocking resources, CSS complexity
- **Optimization:** Reduce server response time, eliminate render-blocking resources, inline critical CSS

### CLS (Cumulative Layout Shift)
- **What affects it:** Images without dimensions, dynamic content injection, web font loading
- **Optimization:** Set image dimensions, reserve space for dynamic content, use font-display

### INP (Interaction to Next Paint)
- **What affects it:** Long JavaScript tasks, heavy DOM manipulation, main thread blocking
- **Optimization:** Break long tasks, use web workers, optimize event handlers

### TTFB (Time to First Byte)
- **What affects it:** Server processing time, network latency, TLS negotiation
- **Optimization:** Server caching, CDN, optimize database queries, HTTP/2
