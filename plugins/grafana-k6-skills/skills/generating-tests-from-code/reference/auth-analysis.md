# Authentication Analysis Reference

## Auth Detection Patterns by Framework

### Express (Node.js)

**JWT Authentication:**
```javascript
// Look for: passport, jsonwebtoken, express-jwt, middleware patterns
const jwt = require('jsonwebtoken');
app.use('/api', authenticateToken);  // Global auth middleware

function authenticateToken(req, res, next) {
  const token = req.headers['authorization']?.split(' ')[1];
  jwt.verify(token, process.env.SECRET, (err, user) => { /* ... */ });
}
```
**Detection:** Search for `jsonwebtoken`, `passport-jwt`, `express-jwt`, `Authorization` header parsing.

**Session Authentication:**
```javascript
// Look for: express-session, passport with session strategy
app.use(session({ secret: '...', resave: false }));
app.use(passport.initialize());
app.use(passport.session());
```
**Detection:** Search for `express-session`, `passport.session()`, `req.session`.

**API Key:**
```javascript
// Look for: custom middleware checking headers or query params
app.use((req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  if (!apiKey || !validKeys.includes(apiKey)) return res.status(401).json({});
});
```
**Detection:** Search for `x-api-key`, `apikey`, `api_key` in middleware.

### NestJS

**Guards (JWT):**
```typescript
// Look for: @UseGuards, AuthGuard, JwtAuthGuard
@UseGuards(JwtAuthGuard)
@Controller('users')
export class UsersController { /* ... */ }

// Or per-route
@UseGuards(AuthGuard('jwt'))
@Get('profile')
getProfile() { /* ... */ }
```
**Detection:** Search for `@UseGuards`, `AuthGuard`, `JwtStrategy`, `@nestjs/passport`.

**Public Routes:**
```typescript
// Look for: @Public(), @SkipAuth(), or custom decorators
@Public()
@Get('health')
healthCheck() { /* ... */ }
```
**Detection:** Search for `@Public`, `@SkipAuth`, `SetMetadata('isPublic', true)`.

### Spring Boot (Java)

**Spring Security:**
```java
// Look for: SecurityFilterChain, WebSecurityConfigurerAdapter
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/public/**").permitAll()
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated()
    );
}
```
**Detection:** Search for `SecurityFilterChain`, `@EnableWebSecurity`, `authorizeHttpRequests`, `antMatchers`.

**JWT in Spring:**
```java
// Look for: OncePerRequestFilter, JWT token parsing
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    protected void doFilterInternal(request, response, chain) {
        String token = request.getHeader("Authorization");
        // verify token
    }
}
```
**Detection:** Search for `JwtAuthFilter`, `OncePerRequestFilter`, `@PreAuthorize`.

### FastAPI (Python)

**OAuth2/JWT:**
```python
# Look for: Depends(), OAuth2PasswordBearer, HTTPBearer
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.get("/users/me")
async def read_users_me(token: str = Depends(oauth2_scheme)):
    user = decode_token(token)
    return user
```
**Detection:** Search for `Depends(`, `OAuth2PasswordBearer`, `HTTPBearer`, `Security(`.

**API Key:**
```python
api_key_header = APIKeyHeader(name="X-API-Key")

@app.get("/data")
async def get_data(api_key: str = Depends(api_key_header)):
    pass
```
**Detection:** Search for `APIKeyHeader`, `APIKeyQuery`, `APIKeyCookie`.

### Django REST Framework (Python)

**Permission Classes:**
```python
# Look for: permission_classes, authentication_classes
class UserViewSet(viewsets.ModelViewSet):
    authentication_classes = [TokenAuthentication, SessionAuthentication]
    permission_classes = [IsAuthenticated]
```
**Detection:** Search for `permission_classes`, `authentication_classes`, `IsAuthenticated`, `AllowAny`, `TokenAuthentication`, `JWTAuthentication`.

---

## k6 Auth Setup Generation

### JWT Token Flow

```javascript
export function setup() {
  const loginRes = http.post(`${BASE_URL}/auth/login`,
    JSON.stringify({
      username: __ENV.TEST_USERNAME || 'testuser',
      password: __ENV.TEST_PASSWORD || 'testpass',
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );

  check(loginRes, { 'login successful': (r) => r.status === 200 });
  const token = loginRes.json('access_token');
  return { token };
}

export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };
  // Use headers in all requests
}
```

### Session/Cookie Flow

```javascript
export default function () {
  // Login (cookies auto-stored in VU cookie jar)
  const loginRes = http.post(`${BASE_URL}/login`, {
    username: 'testuser',
    password: 'testpass',
  });
  check(loginRes, { 'logged in': (r) => r.status === 200 || r.status === 302 });

  // Subsequent requests include session cookie automatically
  http.get(`${BASE_URL}/dashboard`);
}
```

### API Key Flow

```javascript
const API_KEY = __ENV.API_KEY || 'test-api-key';

export default function () {
  http.get(`${BASE_URL}/data`, {
    headers: { 'X-API-Key': API_KEY },
  });
}
```

### OAuth2 Client Credentials Flow

```javascript
import encoding from 'k6/encoding';

export function setup() {
  const res = http.post(`${AUTH_URL}/oauth/token`,
    'grant_type=client_credentials&scope=read write',
    {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': `Basic ${encoding.b64encode(`${CLIENT_ID}:${CLIENT_SECRET}`)}`,
      },
    }
  );
  return { token: res.json('access_token') };
}
```

## Identifying Protected vs Public Endpoints

Build a table of endpoints with their auth requirements:

| Endpoint | Method | Auth Required | Auth Type | Notes |
|----------|--------|---------------|-----------|-------|
| `/health` | GET | No | - | Health check |
| `/auth/login` | POST | No | - | Login endpoint |
| `/api/users` | GET | Yes | JWT Bearer | List users |
| `/api/admin/settings` | GET | Yes | JWT + Admin Role | Admin only |

Use this table to:
1. Generate `setup()` with appropriate login flow
2. Test public endpoints without auth
3. Group protected endpoints under authenticated scenarios
4. Test authorization (access control) with different user roles
