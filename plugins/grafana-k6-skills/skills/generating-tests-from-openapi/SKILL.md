---
name: generating-tests-from-openapi
description: Use when generating k6 load test scripts from OpenAPI specification files (YAML or JSON). Use when the user has an OpenAPI spec, API specification, or wants to create k6 tests from API documentation defined in OpenAPI 3.0 or 3.1 format.
---

# Generating k6 Tests from OpenAPI

Generate k6 load test scripts by parsing OpenAPI specification files (YAML format). Extracts endpoints, parameters, request bodies, and security schemes to create comprehensive test suites.

## k6 Version Compatibility

Written for **k6 v2.3.0** (current stable). The v1.8.x maintenance line is supported where noted; differences are marked `v1.x:` inline. Before generating, determine the version of the environment that will run the test:

1. Run `k6 version` where the test will execute (prints `k6 v2.3.0 (...)`). For Docker, CI, or k6-operator, read the pinned image tag, the `setup-k6-action` `k6-version` input, or `spec.runner.image`.
2. If no version is visible, ask the user. Otherwise assume v2.x and record the assumption in a header comment of the generated script, for example `// Target: k6 v2.3.0`.
3. Apply the notes below when the target is v1.x.

**Version notes for this skill:**
- Map each OpenAPI `tag` to its own scenario so users can run a subset with `k6 run --scenario Users,Orders script.js` (v2.3.0+). `v1.x:` add an `__ENV.SCENARIO` switch.
- Validate with `k6 inspect script.js`, then `k6 run --once script.js` (v2.3.0+). `v1.x:` `k6 run --vus 1 --iterations 1 script.js`.
- `securitySchemes` of type `http`/`basic` become an `Authorization: Basic ...` header or URL credentials; never `auth: 'basic'` (no effect).
- Operations that use the HTTP `QUERY` method map to `http.query()` (documented for k6 v2.3.x); on older versions fall back to `http.request('QUERY', ...)` only if the server accepts it, otherwise skip them with a note.

For the full feature/version matrix see `/k6:operating-k6-in-ci-cd` (reference/version-compatibility.md).

## Workflow

### Step 1: Parse OpenAPI Spec

Read the OpenAPI YAML file and extract:
- API info (title, version, base URL)
- All paths and operations
- Request parameters, bodies, and response schemas
- Security definitions and requirements

### Step 2: Security Scheme Analysis (Prerequisite)

**Before generating test scripts, analyze security requirements.**

1. **Extract security definitions** from `components.securitySchemes`:
   - OAuth2 flows (authorization_code, client_credentials, password)
   - API Key (header, query, cookie)
   - HTTP Bearer / Basic Auth
   - OpenID Connect

2. **Map per-endpoint security:**
   - Check global `security` array
   - Check per-operation `security` overrides
   - Identify public endpoints (empty security array `security: []`)

3. **Generate k6 auth setup:**
   - Create `setup()` function matching the required auth flow
   - Configure token/key handling for requests

See [reference/openapi-auth-mapping.md](reference/openapi-auth-mapping.md) for mapping security schemes to k6 code.

### Step 3: Endpoint Extraction

For each path and operation, extract:

```yaml
paths:
  /users:
    get:                          # HTTP method
      operationId: listUsers      # Operation identifier
      tags: [Users]               # Grouping
      parameters:                 # Query/header/path params
        - name: page
          in: query
          schema:
            type: integer
      security:                   # Per-operation security
        - bearerAuth: []
      responses:
        '200':
          description: Success
```

Map to k6:
- **Path + method** → HTTP request (`get/post/put/patch/del/head/options`, `query` for QUERY on k6 v2.3.x)
- **Parameters** → URL params, query string, headers
- **Request body** → `http.post()` body (`JSON.stringify` + `Content-Type: application/json`; a bare object is form-encoded)
- **Tags** → one k6 scenario per tag (selectable with `--scenario`, v2.3.0+) with groups inside
- **Security** → Auth headers
- **`operationId`** → request `tags: { name: operationId }` so dynamic path parameters do not fragment metrics

### Step 4: Generate Request Bodies

Convert schema definitions to example values:

| Schema Type | Example Value |
|------------|---------------|
| `string` | `"test-string"` |
| `string (email)` | `"test@example.com"` |
| `string (date-time)` | `"2024-01-01T00:00:00Z"` |
| `string (uuid)` | `"550e8400-e29b-41d4-a716-446655440000"` |
| `integer` | `1` |
| `number` | `1.0` |
| `boolean` | `true` |
| `array` | `[<item_example>]` |
| `object` | `{<property_examples>}` |
| `enum` | First enum value |

Use `example` or `default` values from the spec when available.

See [reference/openapi-parsing.md](reference/openapi-parsing.md) for detailed parsing logic.

### Step 5: Script Generation

Organize the generated script by API tags, one scenario per tag:

```javascript
// Target: k6 v2.3.0
import http from 'k6/http';
import { check, group, sleep } from 'k6';

const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';

export const options = {
  scenarios: {
    // Run one tag only: k6 run --scenario users script.js
    users: {
      executor: 'ramping-vus',
      exec: 'users',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 10 },
        { duration: '5m', target: 10 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export function setup() {
  // Auth flow based on security scheme
}

export function users(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('Users', () => {
    // GET /users
    const listRes = http.get(`${BASE_URL}/users?page=1&limit=10`, {
      headers, tags: { name: 'ListUsers' },
    });
    check(listRes, { 'list users 200': (r) => r.status === 200 });

    // POST /users
    const createRes = http.post(`${BASE_URL}/users`,
      JSON.stringify({ name: 'Test User', email: 'test@example.com' }),
      { headers, tags: { name: 'CreateUser' } }
    );
    check(createRes, { 'create user 201': (r) => r.status === 201 });

    // GET /users/{id}
    const userId = createRes.json('id') || 1;
    const getRes = http.get(`${BASE_URL}/users/${userId}`, {
      headers, tags: { name: 'GetUser' },
    });
    check(getRes, { 'get user 200': (r) => r.status === 200 });
  });

  sleep(1);
}
```

### Step 6: Configure Scenarios

Based on endpoint characteristics:
- **Read-heavy endpoints** (GET list) → Higher rate
- **Write endpoints** (POST, PUT) → Lower rate
- **Admin endpoints** → Separate scenario or excluded
- **Public endpoints** → No auth scenario

Because each tag is a scenario, the same script serves every stage: `k6 run --scenario users script.js` while iterating on one resource, `k6 run --scenario users,orders --once script.js` as a smoke check, plain `k6 run script.js` for the full load (v2.3.0+).

### Step 7: Validate

1. `k6 inspect script.js` — parse check and resolved options
2. `k6 run --once script.js` (v2.3.0+) against a dev server, or `k6 run --scenario <tag> --once script.js` for a single tag. `v1.x:` `k6 run --vus 1 --iterations 1 script.js`
3. Compare returned status codes with the `responses` declared in the spec and report mismatches

## OpenAPI Spec Quick Reference

```yaml
openapi: '3.1.0'
info:
  title: My API
  version: '1.0.0'
servers:
  - url: https://api.example.com/v1
security:
  - bearerAuth: []               # Global security
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key
  schemas:
    User:
      type: object
      required: [name, email]
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
paths:
  /users:
    get:
      tags: [Users]
      parameters:
        - name: page
          in: query
          schema: { type: integer, default: 1 }
      responses:
        '200':
          description: User list
    post:
      tags: [Users]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/User'
      responses:
        '201':
          description: Created
  /users/{id}:
    get:
      tags: [Users]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '200':
          description: User details
```

## Related Skills

- For HTTP/gRPC/WebSocket API patterns: `/k6:generating-api-load-tests`
- For scenario and threshold design: `/k6:designing-test-scenarios`
- For code-based test generation: `/k6:generating-tests-from-code`
