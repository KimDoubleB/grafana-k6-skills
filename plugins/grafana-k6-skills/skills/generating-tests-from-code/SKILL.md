---
name: generating-tests-from-code
description: Use when analyzing an existing API codebase to auto-generate k6 load test scripts. Use when the user wants to create k6 tests from source code, route definitions, controllers, or middleware in frameworks like Express, NestJS, Spring Boot, FastAPI, or Django.
context: fork
agent: Explore
---

# Generating k6 Tests from Code

Analyze an existing API codebase to automatically generate k6 load test scripts. Detects endpoints, authentication patterns, request/response shapes, and generates comprehensive test suites.

## k6 Version Compatibility

Written for **k6 v2.3.0** (current stable). The v1.8.x maintenance line is supported where noted; differences are marked `v1.x:` inline. Before generating, determine the version of the environment that will run the test:

1. Run `k6 version` where the test will execute (prints `k6 v2.3.0 (...)`). For Docker, CI, or k6-operator, read the pinned image tag, the `setup-k6-action` `k6-version` input, or `spec.runner.image`. Also look in the repository being analyzed: a Dockerfile, CI workflow, or existing k6 scripts often reveal the version already in use.
2. If no version is visible, ask the user. Otherwise assume v2.x and record the assumption in a header comment of the generated script, for example `// Target: k6 v2.3.0`.
3. Apply the notes below when the target is v1.x.

**Version notes for this skill:**
- Generate one scenario per resource group and let users run subsets with `k6 run --scenario users,orders script.js` (v2.3.0+). `v1.x:` add an `__ENV.SCENARIO` switch when selecting scenarios.
- Validate the generated script with `k6 inspect script.js`, then `k6 run --once script.js` (v2.3.0+) against a dev environment. `v1.x:` `k6 run --vus 1 --iterations 1 script.js`.
- Never emit `auth: 'basic'`; it has no effect. Use an `Authorization` header or credentials in the URL. `'digest'` and `'ntlm'` are the only meaningful `auth` values.
- Use `k6/secrets` (`--secret-source=file=...`) rather than `__ENV` for credentials discovered in the codebase; secret values are redacted from logs.

For the full feature/version matrix see `/k6:operating-k6-in-ci-cd` (reference/version-compatibility.md).

## Workflow

Follow this workflow strictly in order:

### Step 1: Security & Authentication Analysis (Prerequisite)

**Before analyzing routes, always detect the authentication mechanism first.**

1. **Scan for auth middleware/decorators:**
   - Look for authentication middleware, guards, decorators, or filters
   - Identify the auth type: JWT, session, OAuth2, API Key, Basic Auth
   - Determine which endpoints are protected vs public

2. **Extract the authentication flow:**
   - Find login/token endpoints
   - Identify how tokens are obtained, refreshed, and passed
   - Check for CSRF protection patterns

3. **Generate k6 auth setup:**
   - Create `setup()` function for authentication
   - Configure token/session handling for `default()` function
   - Handle token refresh if needed

See [reference/auth-analysis.md](reference/auth-analysis.md) for auth detection patterns per framework.

### Step 2: Route & Controller Analysis

1. **Scan route definitions:**
   - Map all endpoints with HTTP methods
   - Extract URL parameters, query parameters, request body schemas
   - Identify middleware chain per route

2. **Extract request/response shapes:**
   - Analyze request body types/validation
   - Identify required vs optional fields
   - Note content types (JSON, form-data, multipart)

3. **Categorize endpoints:**
   - Group by resource/domain (users, products, orders, etc.)
   - Identify CRUD patterns
   - Note rate-limited or special endpoints

See [reference/framework-patterns.md](reference/framework-patterns.md) for framework-specific analysis patterns.

### Step 3: Scenario Design

1. **Map endpoints to test scenarios:**
   - High-traffic endpoints → higher iteration rate
   - CRUD sequences → grouped in order (create → read → update → delete)
   - Background/admin endpoints → lower priority

2. **Design realistic user flows:**
   - Login → browse → select → purchase
   - Weighted distribution matching production traffic

### Step 4: Script Generation

Generate the k6 test script with:
- A header comment with the target k6 version (for example `// Target: k6 v2.3.0`)
- `setup()` for authentication
- One scenario per resource group (so `--scenario` can run a subset) and grouped endpoint calls inside each `exec` function
- Checks for each response
- Appropriate thresholds, including per-scenario ones (`'http_req_duration{scenario:orders}'`)
- Data parameterization where needed
- Tags for per-endpoint metrics (`tags: { name: 'GetUser' }` to collapse dynamic URLs)

### Step 5: Validate

1. `k6 inspect script.js` — parses the script and prints the resolved options; catches syntax errors and typos in option names
2. `k6 run --once script.js` (v2.3.0+) against a development environment — one iteration with the real scenario settings. `v1.x:` `k6 run --vus 1 --iterations 1 script.js`
3. Report which endpoints returned non-2xx so the user can fix auth or test data before the load run

## Supported Frameworks

### Node.js
- **Express** — `app.get()`, `router.get()`, route files
- **Fastify** — `fastify.get()`, route plugins, schemas
- **NestJS** — `@Controller()`, `@Get()`, `@Post()` decorators, Guards

### Java
- **Spring Boot** — `@RestController`, `@GetMapping`, `@PostMapping`, `@RequestMapping`, Security config

### Python
- **FastAPI** — `@app.get()`, `@router.get()`, Pydantic models, Depends()
- **Django REST Framework** — `ViewSet`, `@api_view`, `urlpatterns`, permissions

## Output Structure

The generated script should follow this structure:

```javascript
// Target: k6 v2.3.0
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// Test data (if applicable)
const testData = new SharedArray('data', function () {
  return JSON.parse(open('./test-data.json'));
});

export const options = {
  scenarios: {
    // one scenario per resource group; run a subset with: k6 run --scenario users script.js
    users: { executor: 'constant-arrival-rate', rate: 50, timeUnit: '1s', duration: '5m', preAllocatedVUs: 20, exec: 'users' },
    orders: { executor: 'constant-arrival-rate', rate: 10, timeUnit: '1s', duration: '5m', preAllocatedVUs: 10, exec: 'orders' },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    'http_req_duration{scenario:users}': ['p(95)<300'],
    'http_req_duration{scenario:orders}': ['p(95)<800'],
  },
};

// Authentication setup
export function setup() {
  // Login and return auth token/session
}

// One exported function per scenario
export function users(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('Users CRUD', () => {
    // CRUD operations for the users resource
  });
}

export function orders(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('Orders CRUD', () => {
    // CRUD operations for the orders resource
  });
}

export function teardown(data) {
  // Cleanup if needed
}
```

## Related Skills

- For HTTP/gRPC/WebSocket API patterns: `/k6:generating-api-load-tests`
- For scenario and threshold design: `/k6:designing-test-scenarios`
- For OpenAPI spec-based generation: `/k6:generating-tests-from-openapi`
