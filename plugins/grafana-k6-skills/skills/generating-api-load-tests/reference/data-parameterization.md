# k6 Data Parameterization Reference

## Overview

Data parameterization allows k6 tests to use external data files, environment variables, and dynamically generated data. This makes tests more realistic and avoids hardcoded values.

## SharedArray

`SharedArray` stores data once in memory and shares it across all VUs. Essential for large datasets to avoid excessive memory usage.

```javascript
import { SharedArray } from 'k6/data';

// Must be created in init context
const users = new SharedArray('users', function () {
  return JSON.parse(open('./users.json'));
});

export default function () {
  // Access by index (read-only)
  const user = users[Math.floor(Math.random() * users.length)];
  console.log(user.name, user.email);
}
```

### SharedArray Rules

- **Must be in init context** (top-level code, not inside default/setup)
- **Load data inside constructor function** — not before it
- **Read-only** — cannot modify after construction
- **Supported operations**: `length`, `[index]`, `for...of` loops
- **Do NOT** use `.filter()`, `.map()`, `.forEach()` on SharedArray (creates regular array, defeats purpose)
- **Do NOT** pass SharedArray from `setup()` to `default()` (gets marshalled)
- **Do NOT** `JSON.stringify()` the entire array (creates regular copy)

```javascript
// CORRECT: Load and filter inside constructor
const activeUsers = new SharedArray('active-users', function () {
  const all = JSON.parse(open('./users.json'));
  return all.filter(u => u.active); // Filter returns regular array, stored as SharedArray
});

// WRONG: Loading outside constructor
const rawData = open('./users.json'); // Loaded for every VU
const users = new SharedArray('users', function () {
  return JSON.parse(rawData); // rawData already duplicated
});
```

## CSV Files

### Using k6/experimental/csv (Native)

> **Note:** These modules are experimental and may change in future k6 versions. Check [k6 release notes](https://grafana.com/docs/k6/latest/release-notes/) for graduation status.

```javascript
import { open } from 'k6/experimental/fs';
import csv from 'k6/experimental/csv';
import { SharedArray } from 'k6/data';

// Full parse into SharedArray
const csvFile = open('./data.csv');
const data = new SharedArray('csv-data', function () {
  return csv.parse(csvFile, { delimiter: ',', skipFirstLine: false });
});

export default function () {
  const row = data[0]; // Access row as object (column headers as keys)
}
```

### Using Papa Parse (jslib)

```javascript
import papaparse from 'https://jslib.k6.io/papaparse/5.1.1/index.js';
import { SharedArray } from 'k6/data';

const csvData = new SharedArray('csv', function () {
  return papaparse.parse(open('./data.csv'), { header: true }).data;
});

export default function () {
  const record = csvData[Math.floor(Math.random() * csvData.length)];
  console.log(record.username, record.password);
}
```

### CSV File Example

```csv
username,password,email
user1,pass1,user1@test.com
user2,pass2,user2@test.com
user3,pass3,user3@test.com
```

## JSON Files

```javascript
import { SharedArray } from 'k6/data';

const testData = new SharedArray('json-data', function () {
  return JSON.parse(open('./test-data.json'));
});

export default function () {
  const item = testData[__ITER % testData.length]; // Sequential per iteration
  console.log(item.name);
}
```

### JSON File Example

```json
[
  { "id": 1, "name": "Item A", "price": 29.99 },
  { "id": 2, "name": "Item B", "price": 49.99 },
  { "id": 3, "name": "Item C", "price": 19.99 }
]
```

## Data Access Patterns

### Random Selection

```javascript
const item = data[Math.floor(Math.random() * data.length)];
```

### Sequential Per Iteration (Across All VUs)

```javascript
import exec from 'k6/execution';

const item = data[exec.scenario.iterationInTest % data.length];
```

### Unique Per VU

```javascript
import exec from 'k6/execution';

// Each VU gets a different data item
const item = data[exec.vu.idInTest - 1]; // VU IDs are 1-based
```

### Sequential Per VU Iteration

```javascript
import exec from 'k6/execution';

// Each VU cycles through data independently
const item = data[exec.vu.iterationInInstance % data.length];
```

### Unique Per Iteration (No Repeats)

```javascript
import exec from 'k6/execution';

// Each iteration gets unique data (requires enough data rows)
const item = data[exec.scenario.iterationInTest];
if (!item) {
  console.warn('Ran out of test data');
  return;
}
```

## Environment Variables

### Passing Variables

```bash
# Via command line
k6 run -e BASE_URL=https://staging.example.com -e API_KEY=abc123 script.js

# Via k6 options environment variable
K6_VUS=10 K6_DURATION=30s k6 run script.js
```

### Accessing Variables

```javascript
const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';
const API_KEY = __ENV.API_KEY;

export default function () {
  http.get(`${BASE_URL}/users`, {
    headers: { 'X-API-Key': API_KEY },
  });
}
```

## Dynamic Data Generation

### Using xk6-faker Extension

```javascript
import faker from 'k6/x/faker';

export default function () {
  const user = {
    name: faker.person.firstName() + ' ' + faker.person.lastName(),
    email: faker.internet.email(),
    phone: faker.phone.number(),
    address: faker.location.streetAddress(),
  };

  http.post('https://api.example.com/users',
    JSON.stringify(user),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```

### Manual Generation

```javascript
function randomString(length) {
  const chars = 'abcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

function randomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

export default function () {
  const payload = {
    username: `user_${randomString(8)}`,
    age: randomInt(18, 65),
    score: Math.random() * 100,
    timestamp: new Date().toISOString(),
  };
}
```

## File Reading with open()

```javascript
// Text file (default)
const textData = open('./data.txt');

// Binary file
const binaryData = open('./image.png', 'b');

// JSON file
const jsonData = JSON.parse(open('./config.json'));

// CSV file (raw text)
const csvRaw = open('./data.csv');
```

**Rules for `open()`:**
- Must be called in init context (top-level code)
- Cannot be called inside `default()`, `setup()`, or `teardown()`
- Paths are relative to the script location
- Use mode `'b'` for binary files (returns ArrayBuffer)

## Complete Example: Parameterized API Test

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';
import exec from 'k6/execution';

const users = new SharedArray('users', function () {
  return JSON.parse(open('./users.json'));
  // Expected format: [{ "username": "user1", "password": "pass1" }, ...]
});

const products = new SharedArray('products', function () {
  return JSON.parse(open('./products.json'));
  // Expected format: [{ "id": 1, "name": "Product A" }, ...]
});

const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';

export const options = {
  scenarios: {
    browse_and_buy: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },
        { duration: '3m', target: 20 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<800'],
    http_req_failed: ['rate<0.05'],
  },
};

export function setup() {
  // Login with first user to verify API is reachable
  const res = http.post(`${BASE_URL}/auth/login`,
    JSON.stringify(users[0]),
    { headers: { 'Content-Type': 'application/json' } }
  );
  check(res, { 'login works': (r) => r.status === 200 });
}

export default function () {
  // Each VU uses a different user
  const user = users[exec.vu.idInTest % users.length];

  // Login
  const loginRes = http.post(`${BASE_URL}/auth/login`,
    JSON.stringify(user),
    { headers: { 'Content-Type': 'application/json' } }
  );
  const token = loginRes.json('token');
  const authHeaders = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  };

  // Browse random product
  const product = products[Math.floor(Math.random() * products.length)];
  const productRes = http.get(`${BASE_URL}/products/${product.id}`, {
    headers: authHeaders,
    tags: { name: 'GetProduct' },
  });
  check(productRes, { 'product loaded': (r) => r.status === 200 });

  // Add to cart
  const cartRes = http.post(`${BASE_URL}/cart/items`,
    JSON.stringify({ productId: product.id, quantity: 1 }),
    { headers: authHeaders, tags: { name: 'AddToCart' } }
  );
  check(cartRes, { 'added to cart': (r) => r.status === 201 });

  sleep(Math.random() * 2 + 1); // Realistic think time
}
```
