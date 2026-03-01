# OpenAPI Security Scheme to k6 Mapping

## Security Scheme Types

### HTTP Bearer

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT    # Optional, informational
```

**k6 mapping:**
```javascript
export function setup() {
  // Obtain token (login endpoint must be identified separately)
  const res = http.post(`${BASE_URL}/auth/login`,
    JSON.stringify({ username: __ENV.USERNAME, password: __ENV.PASSWORD }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  return { token: res.json('access_token') };
}

export default function (data) {
  http.get(`${BASE_URL}/resource`, {
    headers: { 'Authorization': `Bearer ${data.token}` },
  });
}
```

### HTTP Basic

```yaml
components:
  securitySchemes:
    basicAuth:
      type: http
      scheme: basic
```

**k6 mapping:**
```javascript
import encoding from 'k6/encoding';

const credentials = encoding.b64encode(
  `${__ENV.USERNAME}:${__ENV.PASSWORD}`
);

export default function () {
  http.get(`${BASE_URL}/resource`, {
    headers: { 'Authorization': `Basic ${credentials}` },
  });
}
```

### API Key (Header)

```yaml
components:
  securitySchemes:
    apiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
```

**k6 mapping:**
```javascript
const API_KEY = __ENV.API_KEY;

export default function () {
  http.get(`${BASE_URL}/resource`, {
    headers: { 'X-API-Key': API_KEY },
  });
}
```

### API Key (Query)

```yaml
components:
  securitySchemes:
    apiKeyQuery:
      type: apiKey
      in: query
      name: api_key
```

**k6 mapping:**
```javascript
const API_KEY = __ENV.API_KEY;

export default function () {
  http.get(`${BASE_URL}/resource?api_key=${API_KEY}`);
}
```

### API Key (Cookie)

```yaml
components:
  securitySchemes:
    apiKeyCookie:
      type: apiKey
      in: cookie
      name: session_id
```

**k6 mapping:**
```javascript
export default function () {
  http.get(`${BASE_URL}/resource`, {
    cookies: { session_id: __ENV.SESSION_ID },
  });
}
```

### OAuth2 — Client Credentials

```yaml
components:
  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://auth.example.com/oauth/token
          scopes:
            read: Read access
            write: Write access
```

**k6 mapping:**
```javascript
import encoding from 'k6/encoding';

const TOKEN_URL = 'https://auth.example.com/oauth/token';
const CLIENT_ID = __ENV.CLIENT_ID;
const CLIENT_SECRET = __ENV.CLIENT_SECRET;

export function setup() {
  const res = http.post(TOKEN_URL,
    'grant_type=client_credentials&scope=read write',
    {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': `Basic ${encoding.b64encode(`${CLIENT_ID}:${CLIENT_SECRET}`)}`,
      },
    }
  );
  check(res, { 'token obtained': (r) => r.status === 200 });
  return { token: res.json('access_token') };
}

export default function (data) {
  http.get(`${BASE_URL}/resource`, {
    headers: { 'Authorization': `Bearer ${data.token}` },
  });
}
```

### OAuth2 — Password (Resource Owner)

```yaml
components:
  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        password:
          tokenUrl: https://auth.example.com/oauth/token
          scopes:
            read: Read access
```

**k6 mapping:**
```javascript
export function setup() {
  const res = http.post(TOKEN_URL,
    `grant_type=password&username=${__ENV.USERNAME}&password=${__ENV.PASSWORD}&scope=read`,
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );
  return { token: res.json('access_token') };
}
```

### OAuth2 — Authorization Code

Not directly applicable for automated load testing since it requires browser-based user interaction. Alternatives:
1. Use a pre-obtained token via environment variable
2. Use the password flow if the auth server supports it
3. Use client credentials flow for service-to-service testing

```javascript
// Pre-obtained token approach
const TOKEN = __ENV.OAUTH_TOKEN;

export default function () {
  http.get(`${BASE_URL}/resource`, {
    headers: { 'Authorization': `Bearer ${TOKEN}` },
  });
}
```

### OpenID Connect

```yaml
components:
  securitySchemes:
    openId:
      type: openIdConnect
      openIdConnectUrl: https://auth.example.com/.well-known/openid-configuration
```

**k6 mapping:** Similar to OAuth2. Fetch the OpenID configuration to discover the token endpoint, then use the appropriate OAuth2 flow.

## Per-Endpoint Security

```yaml
security:
  - bearerAuth: []    # Global default

paths:
  /public/health:
    get:
      security: []    # No auth required (overrides global)

  /users:
    get:
      security:
        - bearerAuth: []
        - apiKeyAuth: []  # Either bearer OR API key

  /admin/settings:
    put:
      security:
        - bearerAuth: [admin]  # Bearer with admin scope
```

**k6 mapping strategy:**
1. Public endpoints → No auth headers
2. Multiple schemes (OR) → Use the simplest one for testing
3. Scoped access → Ensure test credentials have required scopes
4. Generate separate scenarios for different auth levels if needed
