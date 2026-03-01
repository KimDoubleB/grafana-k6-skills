# k6 HTTP API Reference

## HTTP Methods

```javascript
import http from 'k6/http';

http.get(url, [params])                    // GET request
http.post(url, [body], [params])           // POST request
http.put(url, [body], [params])            // PUT request
http.patch(url, [body], [params])          // PATCH request
http.del(url, [body], [params])            // DELETE request
http.head(url, [params])                   // HEAD request
http.options(url, [body], [params])        // OPTIONS request
http.request(method, url, [body], [params]) // Any HTTP method
http.asyncRequest(method, url, [body], [params]) // Async (returns Promise)
```

## Request Parameters (Params Object)

| Parameter | Type | Description |
|-----------|------|-------------|
| `auth` | string | `'basic'`, `'digest'`, or `'ntlm'` |
| `cookies` | object | Request-scoped cookies `{ name: 'value' }` or `{ name: { value: 'v', replace: true } }` |
| `headers` | object | HTTP headers `{ 'Content-Type': 'application/json' }` |
| `jar` | CookieJar | Override default VU cookie jar |
| `redirects` | number | Max redirects to follow (default: follow all) |
| `tags` | object | Custom metric tags `{ name: 'MyRequest' }` |
| `timeout` | string/number | Max wait time (default: `'60s'`) |
| `compression` | string | `'gzip'`, `'deflate'`, `'br'`, `'zstd'` or combinations |
| `responseType` | string | `'text'` (default), `'binary'`, or `'none'` |
| `responseCallback` | function | Custom response callback (e.g., `http.expectedStatuses(200, 201)`) |

## Response Object

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `status` | number | HTTP status code |
| `statusText` | string | HTTP status text |
| `body` | string | Response body |
| `headers` | object | Response headers (canonical key format) |
| `cookies` | object | Response cookies with metadata |
| `error` | string | Error message (non-HTTP errors) |
| `error_code` | number | Error code |
| `proto` | string | HTTP protocol (`'HTTP/1.0'`, `'HTTP/1.1'`, `'HTTP/2.0'`) |
| `remote_ip` | string | Server IP address |
| `remote_port` | number | Server port number |
| `url` | string | Final URL (after redirects) |
| `request` | object | Request details (body, cookies, headers, method, url) |
| `timings` | object | See Timings below |
| `tls_version` | string | TLS version (e.g., `'tls1.2'`) |
| `tls_cipher_suite` | string | Cipher suite used |

### Timings

| Property | Description |
|----------|-------------|
| `timings.blocked` | Time spent blocked before starting request (DNS lookup, connection queue) |
| `timings.connecting` | Time spent establishing TCP connection |
| `timings.tls_handshaking` | Time spent on TLS handshake |
| `timings.sending` | Time spent sending request body |
| `timings.waiting` | Time spent waiting for server response (TTFB) |
| `timings.receiving` | Time spent receiving response body |
| `timings.duration` | Total request time (sending + waiting + receiving) |

### Methods

```javascript
res.json()                    // Parse body as JSON
res.json('path.to.field')     // Parse JSON with gjson selector
res.html()                    // Parse body as HTML (returns Selection)
res.submitForm({              // Submit an HTML form
  formSelector: 'form',
  fields: { username: 'test' },
  submitSelector: '[type="submit"]',
  params: {},
})
res.clickLink({ selector: 'a' })  // Click a link in HTML response
```

## Authentication Patterns

### Basic Auth

```javascript
// Method 1: Using auth param
const res = http.get('https://api.example.com/protected', {
  headers: {
    Authorization: 'Basic ' + encoding.b64encode('user:pass'),
  },
});

// Method 2: Using built-in auth
const res = http.get('https://user:pass@api.example.com/protected', {
  auth: 'basic',
});
```

### Bearer Token

```javascript
export function setup() {
  const loginRes = http.post('https://api.example.com/auth/login',
    JSON.stringify({ username: 'user', password: 'pass' }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  return { token: loginRes.json('access_token') };
}

export default function (data) {
  const params = {
    headers: { Authorization: `Bearer ${data.token}` },
  };
  http.get('https://api.example.com/protected', params);
}
```

### OAuth 2.0 (Client Credentials)

```javascript
import encoding from 'k6/encoding';

export function setup() {
  const res = http.post('https://auth.example.com/oauth/token',
    'grant_type=client_credentials&scope=read',
    {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        Authorization: `Basic ${encoding.b64encode('client_id:client_secret')}`,
      },
    }
  );
  return { token: res.json('access_token') };
}

export default function (data) {
  http.get('https://api.example.com/resource', {
    headers: { Authorization: `Bearer ${data.token}` },
  });
}
```

### API Key

```javascript
// In header
http.get('https://api.example.com/data', {
  headers: { 'X-API-Key': 'your-api-key-here' },
});

// In query parameter
http.get('https://api.example.com/data?api_key=your-api-key-here');
```

### Cookie-Based Session

Note: `setup()` runs in a separate VU, so cookies set during `setup()` do **not** transfer to VUs running `default()`. For cookie-based auth, login inside `default()`:

```javascript
export default function () {
  // Login per VU — cookies are stored in this VU's cookie jar
  const loginRes = http.post('https://app.example.com/login', {
    username: 'user',
    password: 'pass',
  });

  // Subsequent requests in this VU automatically include session cookies
  http.get('https://app.example.com/dashboard');
}
```

## CRUD Patterns

```javascript
import http from 'k6/http';
import { check } from 'k6';

const BASE_URL = 'https://api.example.com';
const HEADERS = { 'Content-Type': 'application/json' };

export default function () {
  // CREATE
  const createRes = http.post(`${BASE_URL}/items`,
    JSON.stringify({ name: 'Test Item', value: 42 }),
    { headers: HEADERS }
  );
  check(createRes, { 'created': (r) => r.status === 201 });
  const itemId = createRes.json('id');

  // READ
  const getRes = http.get(`${BASE_URL}/items/${itemId}`);
  check(getRes, { 'fetched': (r) => r.status === 200 });

  // UPDATE
  const updateRes = http.put(`${BASE_URL}/items/${itemId}`,
    JSON.stringify({ name: 'Updated Item', value: 99 }),
    { headers: HEADERS }
  );
  check(updateRes, { 'updated': (r) => r.status === 200 });

  // DELETE
  const deleteRes = http.del(`${BASE_URL}/items/${itemId}`);
  check(deleteRes, { 'deleted': (r) => r.status === 204 });
}
```

## Batch Requests

Send multiple requests in parallel:

```javascript
// Array format
const responses = http.batch([
  ['GET', 'https://api.example.com/users'],
  ['GET', 'https://api.example.com/products'],
  ['POST', 'https://api.example.com/logs', JSON.stringify({ event: 'test' }), { headers: HEADERS }],
]);

// Named object format (easier to reference results)
const responses = http.batch({
  users: { method: 'GET', url: 'https://api.example.com/users' },
  products: { method: 'GET', url: 'https://api.example.com/products' },
});
check(responses.users, { 'users ok': (r) => r.status === 200 });
check(responses.products, { 'products ok': (r) => r.status === 200 });
```

## File Uploads

### Simple File Upload

```javascript
import http from 'k6/http';

const file = open('test-file.png', 'b'); // Binary mode, in init context

export default function () {
  const res = http.post('https://api.example.com/upload', {
    file: http.file(file, 'test-file.png', 'image/png'),
  });
}
```

### Multipart Form Data

```javascript
const imgFile = open('photo.jpg', 'b');
const docFile = open('report.pdf', 'b');

export default function () {
  const res = http.post('https://api.example.com/upload', {
    profile_photo: http.file(imgFile, 'photo.jpg', 'image/jpeg'),
    document: http.file(docFile, 'report.pdf', 'application/pdf'),
    description: 'Test upload',
  });
}
```

## Correlation (Extracting Dynamic Values)

### From JSON Response

```javascript
const res = http.get('https://api.example.com/session');
const sessionId = res.json('data.session_id');
const items = res.json('data.items.#.id'); // Array of IDs (gjson syntax)

http.get(`https://api.example.com/data?session=${sessionId}`);
```

### From HTML Response

```javascript
const res = http.get('https://app.example.com/form');
const doc = res.html();
const csrfToken = doc.find('input[name="csrf_token"]').attr('value');

http.post('https://app.example.com/submit', {
  csrf_token: csrfToken,
  data: 'test',
});
```

### From Headers

```javascript
const res = http.post('https://api.example.com/resource', payload);
const location = res.headers['Location'];
http.get(location); // Follow redirect manually
```

## Cookie Handling

### Manual Cookie Jar

```javascript
const jar = http.cookieJar();

// Set cookies
jar.set('https://api.example.com', 'session', 'abc123', {
  domain: 'api.example.com',
  path: '/',
  secure: true,
  max_age: 3600,
});

// Get cookies
const cookies = jar.cookiesForURL('https://api.example.com');
console.log(cookies.session); // ['abc123']

// Clear cookies
jar.clear('https://api.example.com');

// Delete specific cookie
jar.delete('https://api.example.com', 'session');
```

### Per-Request Cookies

```javascript
http.get('https://api.example.com/data', {
  cookies: {
    session: 'abc123',
    preference: { value: 'dark', replace: true }, // replace VU jar cookie
  },
});
```

## HTML Form Submission

```javascript
// Load the page with the form
const res = http.get('https://app.example.com/login');

// Submit the form
const submitRes = res.submitForm({
  formSelector: 'form#login',
  fields: {
    username: 'testuser',
    password: 'testpass',
  },
  submitSelector: 'button[type="submit"]',
});
```

## Expected Statuses

```javascript
// Mark specific statuses as expected (affects http_req_failed metric)
const res = http.get('https://api.example.com/maybe-404', {
  responseCallback: http.expectedStatuses(200, 404),
});

// Range of statuses
http.get(url, {
  responseCallback: http.expectedStatuses({ min: 200, max: 299 }),
});

// Set globally
http.setResponseCallback(http.expectedStatuses({ min: 200, max: 399 }));
```

## URL Grouping with Name Tag

Prevent dynamic URLs from creating separate metric entries:

```javascript
// Without grouping: each user ID creates a separate metric
http.get(`https://api.example.com/users/${userId}`);

// With grouping: all requests grouped under one metric
http.get(`https://api.example.com/users/${userId}`, {
  tags: { name: 'GetUser' },
});

// Using http.url template literal (auto-groups)
http.get(http.url`https://api.example.com/users/${userId}`);
```
