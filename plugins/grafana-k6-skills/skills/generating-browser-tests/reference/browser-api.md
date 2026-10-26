# k6 Browser API Reference

## Page API

### Navigation

```javascript
await page.goto(url, {
  waitUntil: 'networkidle',  // 'domcontentloaded' | 'load' | 'networkidle'
  timeout: 30000,
  referer: 'https://referrer.example.com',
});

// Start waiting BEFORE the action that triggers navigation
await Promise.all([
  page.waitForNavigation({ waitUntil: 'load', timeout: 30000 }),
  page.locator('a.next').click(),
]);

// Wait for a specific URL (string or RegExp)
await Promise.all([
  page.waitForNavigation({ url: /\/checkout$/ }),
  page.getByRole('button', { name: 'Checkout' }).click(),
]);

await page.waitForLoadState('networkidle');  // 'domcontentloaded' | 'load' | 'networkidle'

await page.goBack();
await page.goForward();
await page.reload();
```

### Page Information

```javascript
page.url();                    // Current URL (sync)
await page.title();            // Page title
await page.content();          // Full HTML content
```

### Screenshots

```javascript
await page.screenshot({
  path: 'screenshot.png',
  fullPage: true,              // false = viewport only
  type: 'png',                 // 'png' | 'jpeg'
  quality: 80,                 // JPEG quality (0-100)
  omitBackground: false,       // Transparent background
  clip: { x: 0, y: 0, width: 800, height: 600 },  // Clipping region
});
```

### Element Selection

```javascript
// CSS selector
page.locator('div.container > button.primary')

// Semantic selectors (recommended)
page.getByRole('button', { name: 'Submit' })
page.getByLabel('Email address')
page.getByPlaceholder('Search...')
page.getByText('Welcome back')
page.getByTestId('login-form')
page.getByAltText('Company logo')
page.getByTitle('Settings')
```

### Executing JavaScript

```javascript
const result = await page.evaluate(() => {
  return {
    width: document.documentElement.clientWidth,
    height: document.documentElement.clientHeight,
    userAgent: navigator.userAgent,
  };
});

// Pass arguments
const text = await page.evaluate((selector) => {
  return document.querySelector(selector).textContent;
}, '.header');
```

### Event Handlers

```javascript
page.on('console', (msg) => console.log('PAGE LOG:', msg.text()));
page.on('request', (req) => console.log('REQUEST:', req.url()));
page.on('response', (res) => console.log('RESPONSE:', res.url(), res.status()));
page.on('requestfailed', (req) => console.log('FAILED:', req.url()));
```

Since k6 v2.2.0 the response header accessors (`response.allHeaders()`, `headerValue()`, `headerValues()`, `headersArray()`) return the raw wire headers for each hop of a redirect chain, including `Set-Cookie`; `headerValue()` matches names case-insensitively.

### Timeouts

```javascript
page.setDefaultTimeout(10000);              // All operations (ms)
page.setDefaultNavigationTimeout(30000);    // Navigation only (ms)
```

---

## Locator API

### Interaction Methods

| Method | Description | Returns |
|--------|-------------|---------|
| `click([options])` | Click element | `Promise<void>` |
| `dblclick([options])` | Double-click | `Promise<void>` |
| `fill(value, [options])` | Fill input/textarea | `Promise<void>` |
| `type(text, [options])` | Type with key events | `Promise<void>` |
| `press(key, [options])` | Press key combo | `Promise<void>` |
| `selectOption(values, [options])` | Select dropdown | `Promise<string[]>` |
| `check([options])` | Check checkbox | `Promise<void>` |
| `uncheck([options])` | Uncheck checkbox | `Promise<void>` |
| `hover([options])` | Hover over element | `Promise<void>` |
| `tap([options])` | Touch tap | `Promise<void>` |

Click options: `button` ('left'|'middle'|'right'), `clickCount`, `delay`, `force`, `modifiers` (['Alt','Control','Meta','Shift']), `timeout`

### State Queries

| Method | Returns | Description |
|--------|---------|-------------|
| `isVisible()` | `Promise<bool>` | Element is visible |
| `isHidden()` | `Promise<bool>` | Element is hidden |
| `isEnabled([options])` | `Promise<bool>` | Element is enabled |
| `isDisabled([options])` | `Promise<bool>` | Element is disabled |
| `isEditable([options])` | `Promise<bool>` | Element is editable |
| `isChecked([options])` | `Promise<bool>` | Checkbox/radio is checked |
| `isInViewport([{ ratio, timeout }])` | `Promise<bool>` | Element intersects the viewport; `ratio` 0–1 sets how much must be visible (default `0`). k6 v2.1.0+ |

### Content Queries

| Method | Returns | Description |
|--------|---------|-------------|
| `textContent([options])` | `Promise<string\|null>` | element.textContent |
| `innerText([options])` | `Promise<string>` | element.innerText |
| `innerHTML([options])` | `Promise<string>` | element.innerHTML |
| `inputValue([options])` | `Promise<string>` | Input value |
| `getAttribute(name, [options])` | `Promise<string\|null>` | Attribute value |

### Collection Methods

```javascript
await locator.count();        // Number of matching elements
locator.first();              // First element (Locator)
locator.last();               // Last element (Locator)
locator.nth(2);               // 3rd element, zero-indexed (Locator)
await locator.all();          // All elements (Locator[])
```

### Filtering

```javascript
locator.filter({ hasText: 'Active' });         // Filter by text
locator.filter({ hasNotText: 'Disabled' });    // Exclude by text
```

### Waiting

```javascript
await locator.waitFor({
  state: 'visible',            // 'attached' | 'visible' | 'hidden' | 'stable'
  timeout: 5000,
});
```

---

## Browser Types: `chromium.connectOverCDP()` (v2.2.0+)

```javascript
import { chromium } from 'k6/browser';

export default async function () {
  const browser = await chromium.connectOverCDP('ws://localhost:9222/devtools/browser/<id>');
  const page = await browser.newPage();
  try {
    await page.goto('https://quickpizza.grafana.com/');
  } finally {
    await page.close();
    await browser.close(); // optional: k6 closes the connection at iteration end
  }
}
```

| Method | Returns | Notes |
|--------|---------|-------|
| `chromium.connectOverCDP(wsEndpoint)` | `Promise<Browser>` | `ws://` or `wss://` CDP endpoint. No `options.browser` scenario block needed. `K6_BROWSER_TIMEOUT` and `K6_BROWSER_DEBUG` still apply |
| `browser.close()` | `Promise<void>` | Only on browsers returned by `connectOverCDP()`. Closes contexts and pages, releases the connection; the external browser keeps running. Using the browser afterwards throws |

Start a local Chrome with `--remote-debugging-port=9222` and read `webSocketDebuggerUrl` from `http://localhost:9222/json/version`. `v1.x:` use the `K6_BROWSER_WS_URL` environment variable with a regular browser scenario.

## BrowserContext API

### Creating Contexts

```javascript
const context = await browser.newContext();
const page = await context.newPage();

// Or directly
const defaultPage = await browser.newPage();  // Uses default context

// Route the context's traffic through a proxy (k6 v2.1.0+)
const proxied = await browser.newContext({
  proxy: {
    server: 'http://proxy.test:8080',   // required when proxy is set
    bypass: 'localhost,127.0.0.1',      // comma-separated hosts that skip the proxy
  },
});
```

`v1.x:` no context-level proxy; launch Chromium with `K6_BROWSER_ARGS='proxy-server=proxy.test:8080'`.

### Cookie Management

```javascript
// Add cookies
await context.addCookies([{
  name: 'session',
  value: 'abc123',
  domain: '.example.com',
  path: '/',
  secure: true,
  httpOnly: true,
  sameSite: 'Lax',
  expires: Date.now() / 1000 + 3600,
}]);

// Get cookies
const cookies = await context.cookies();
const siteCookies = await context.cookies(['https://example.com']);

// Clear cookies
await context.clearCookies();
```

### Permissions and Geolocation

```javascript
await context.grantPermissions(['geolocation']);
await context.setGeolocation({ latitude: 37.7749, longitude: -122.4194 });
await context.clearPermissions();
```

### Offline Mode

```javascript
await context.setOffline(true);   // Simulate offline
await context.setOffline(false);  // Back online
```

### Init Scripts

```javascript
await context.addInitScript('window.__TEST_MODE__ = true;');
```

---

## Keyboard API

```javascript
await page.keyboard.type('Hello World', { delay: 100 });
await page.keyboard.press('Enter');
await page.keyboard.press('Control+a');
await page.keyboard.down('Shift');
await page.keyboard.press('ArrowDown');
await page.keyboard.up('Shift');
await page.keyboard.insertText('Pasted text');
```

## Mouse API

```javascript
await page.mouse.click(100, 200);
await page.mouse.dblclick(100, 200);
await page.mouse.move(100, 200, { steps: 10 });  // Smooth move
await page.mouse.down();
await page.mouse.move(300, 400);
await page.mouse.up();  // Drag effect
```

## Touchscreen API

```javascript
await page.touchscreen.tap(100, 200);
```

---

## Browser Module Options (Environment Variables)

| Variable | Purpose |
|----------|---------|
| `K6_BROWSER_HEADLESS` | `true` (default) runs without a GUI; set `false` to watch the browser locally |
| `K6_BROWSER_ARGS` | Extra Chromium flags, comma-separated, **without** the `--` prefix. Common: `no-sandbox` (root in Docker/CI/Kubernetes), `ignore-certificate-errors`, `disable-web-security`, `proxy-server=host:port` |
| `K6_BROWSER_IGNORE_DEFAULT_ARGS` | Drop specific default Chromium arguments |
| `K6_BROWSER_EXECUTABLE_PATH` | Absolute path to a Chromium/Chrome binary |
| `K6_BROWSER_TIMEOUT` | Timeout for connecting to the browser instance (`30s` default) |
| `K6_BROWSER_DEBUG` | `true` logs all CDP messages |
| `K6_BROWSER_TRACES_METADATA` | Extra key-value attributes on browser trace spans |

```bash
K6_BROWSER_ARGS='ignore-certificate-errors,proxy-server=127.0.0.1:8080' k6 run script.js
docker run --rm -i -e K6_BROWSER_ARGS='no-sandbox' grafana/k6:2.3.0-with-browser run - <script.js
```
