# OpenAPI Spec Parsing Reference

## Spec Structure Overview

```yaml
openapi: '3.1.0'
info: { title, version, description }
servers: [{ url, description }]
security: [{ schemeName: [] }]          # Global security
paths:
  /resource:
    get/post/put/patch/delete:
      operationId: string
      tags: [string]
      summary: string
      parameters: [Parameter]
      requestBody: RequestBody
      responses: { statusCode: Response }
      security: [{ schemeName: [scope] }]  # Per-operation override
components:
  schemas: { SchemaName: Schema }
  securitySchemes: { SchemeName: SecurityScheme }
  parameters: { ParameterName: Parameter }
  requestBodies: { BodyName: RequestBody }
```

## Extracting Paths and Operations

For each entry in `paths`:

1. **Path** → URL path (e.g., `/users/{id}`)
2. **HTTP method** → `get`, `post`, `put`, `patch`, `delete`
3. **Operation** → The operation object under the method

### Path Parameters

```yaml
parameters:
  - name: id
    in: path            # path | query | header | cookie
    required: true
    schema:
      type: integer
```

Map to k6:
- `in: path` → Replace in URL: `/users/${id}`
- `in: query` → Append: `?page=1&limit=10`
- `in: header` → Add to headers: `{ 'X-Custom': 'value' }`

### Request Body

```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateUser'
    multipart/form-data:
      schema:
        type: object
        properties:
          file:
            type: string
            format: binary
```

Map to k6:
- `application/json` → `JSON.stringify(body)` with `Content-Type` header
- `multipart/form-data` → Use `http.file()` for binary fields
- `application/x-www-form-urlencoded` → URL-encoded string body

## Resolving $ref References

```yaml
# Reference
$ref: '#/components/schemas/User'

# Resolves to the object at components.schemas.User
```

Follow `$ref` chains:
1. Parse the JSON pointer path
2. Navigate to the target object
3. Replace the `$ref` with the resolved object
4. Handle circular references (stop at 2 levels deep)

## Schema to Example Value Generation

### Primitive Types

| Schema | Generated Example |
|--------|------------------|
| `type: string` | `"test-string"` |
| `type: string, format: email` | `"test@example.com"` |
| `type: string, format: date` | `"2024-01-15"` |
| `type: string, format: date-time` | `"2024-01-15T10:30:00Z"` |
| `type: string, format: uuid` | `"550e8400-e29b-41d4-a716-446655440000"` |
| `type: string, format: uri` | `"https://example.com"` |
| `type: string, format: password` | `"password123"` |
| `type: string, enum: [A, B, C]` | `"A"` (first value) |
| `type: string, minLength: 5` | `"test-string"` (at least minLength) |
| `type: integer` | `1` |
| `type: integer, minimum: 10` | `10` |
| `type: number` | `1.0` |
| `type: boolean` | `true` |

### Complex Types

```yaml
# Object
type: object
required: [name, email]
properties:
  name: { type: string }
  email: { type: string, format: email }
  age: { type: integer }
# → { "name": "test-string", "email": "test@example.com", "age": 1 }

# Array
type: array
items: { type: string }
# → ["test-string"]

# Array of objects
type: array
items:
  $ref: '#/components/schemas/User'
# → [{ ...user example }]
```

### Using Existing Examples

Prefer spec-provided examples over generated ones:

```yaml
schema:
  type: object
  properties:
    name: { type: string }
  example:
    name: "John Doe"
# Use the example value: { "name": "John Doe" }
```

Check in order:
1. `example` field on the schema
2. `examples` field on the media type
3. `default` field on the schema
4. Generate from type/format

## Response Status Codes

Map response codes to k6 checks:

```yaml
responses:
  '200': { description: Success }
  '201': { description: Created }
  '204': { description: No Content }
  '400': { description: Bad Request }
  '401': { description: Unauthorized }
  '404': { description: Not Found }
```

For primary success responses, generate checks:
- `2xx` responses → `check(res, { 'status 200': (r) => r.status === 200 })`
- Use `http.expectedStatuses()` if testing error scenarios

## Grouping by Tags

```yaml
paths:
  /users:
    get:
      tags: [Users]
  /products:
    get:
      tags: [Products]
```

Map tags to k6 `group()`:

```javascript
group('Users', () => {
  // All operations tagged 'Users'
});
group('Products', () => {
  // All operations tagged 'Products'
});
```

Operations without tags → group as `'Default'` or by path prefix.
