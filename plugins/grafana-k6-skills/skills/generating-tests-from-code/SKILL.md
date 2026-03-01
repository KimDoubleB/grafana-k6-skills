---
name: generating-tests-from-code
description: Use when analyzing an existing API codebase to auto-generate k6 load test scripts. Use when the user wants to create k6 tests from source code, route definitions, controllers, or middleware in frameworks like Express, NestJS, Spring Boot, FastAPI, or Django.
context: fork
agent: Explore
---

# Generating k6 Tests from Code

Analyze an existing API codebase to automatically generate k6 load test scripts. Detects endpoints, authentication patterns, request/response shapes, and generates comprehensive test suites.

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
- `setup()` for authentication
- `default()` with grouped endpoint calls
- Checks for each response
- Appropriate thresholds
- Data parameterization where needed
- Tags for per-endpoint metrics

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
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// Test data (if applicable)
const testData = new SharedArray('data', function () {
  return JSON.parse(open('./test-data.json'));
});

export const options = {
  scenarios: { /* ... */ },
  thresholds: { /* ... */ },
};

// Authentication setup
export function setup() {
  // Login and return auth token/session
}

// Main test function
export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('Resource A', () => {
    // CRUD operations for Resource A
  });

  group('Resource B', () => {
    // CRUD operations for Resource B
  });

  sleep(1);
}

export function teardown(data) {
  // Cleanup if needed
}
```

## Related Skills

- For HTTP/gRPC/WebSocket API patterns: `/k6:generating-api-load-tests`
- For scenario and threshold design: `/k6:designing-test-scenarios`
- For OpenAPI spec-based generation: `/k6:generating-tests-from-openapi`
