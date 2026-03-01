# Framework Route Analysis Patterns

## Express (Node.js)

### Where to Find Routes

```
src/routes/         — Route files
src/app.js          — Main app with app.use() mounts
src/index.js        — Entry point
src/controllers/    — Controller functions
```

### Route Patterns

```javascript
// Direct route definition
app.get('/users', getUsers);
app.post('/users', createUser);
app.put('/users/:id', updateUser);
app.delete('/users/:id', deleteUser);

// Router module
const router = express.Router();
router.get('/', listItems);
router.post('/', createItem);
router.get('/:id', getItem);

// Mount point
app.use('/api/items', router);
// → Full paths: GET /api/items, POST /api/items, GET /api/items/:id
```

### Extracting Info

- **URL params:** `:id`, `:userId` → dynamic path segments
- **Query params:** Look for `req.query.page`, `req.query.limit`
- **Body:** Look for `req.body.name`, `req.body.email` or validation schemas
- **Middleware:** `app.use(auth)`, `router.use(validateBody)` → auth/validation requirements

---

## NestJS

### Where to Find Routes

```
src/modules/*/       — Feature modules
  *.controller.ts    — Controllers with decorators
  *.service.ts       — Business logic
  *.dto.ts           — Request/Response DTOs
  *.module.ts        — Module definitions
src/app.module.ts    — Root module
```

### Route Patterns

```typescript
@Controller('users')         // Base path: /users
export class UsersController {
  @Get()                     // GET /users
  findAll() {}

  @Get(':id')                // GET /users/:id
  findOne(@Param('id') id: string) {}

  @Post()                    // POST /users
  create(@Body() dto: CreateUserDto) {}

  @Put(':id')                // PUT /users/:id
  update(@Param('id') id: string, @Body() dto: UpdateUserDto) {}

  @Delete(':id')             // DELETE /users/:id
  remove(@Param('id') id: string) {}
}
```

### Extracting Info

- **Path prefix:** `@Controller('prefix')` decorator
- **Global prefix:** Check `app.setGlobalPrefix('api')` in main.ts
- **DTOs:** `CreateUserDto`, `UpdateUserDto` → request body shape
- **Params:** `@Param()`, `@Query()`, `@Body()` decorators
- **Guards:** `@UseGuards()` → authentication requirements
- **Pipes:** `@UsePipes(ValidationPipe)` → validation rules

---

## Spring Boot (Java)

### Where to Find Routes

```
src/main/java/**/controller/    — Controller classes
src/main/java/**/config/        — Security config
src/main/resources/             — application.yml (context path)
```

### Route Patterns

```java
@RestController
@RequestMapping("/api/users")      // Base path
public class UserController {

    @GetMapping                     // GET /api/users
    public List<User> findAll() {}

    @GetMapping("/{id}")            // GET /api/users/{id}
    public User findById(@PathVariable Long id) {}

    @PostMapping                    // POST /api/users
    public User create(@RequestBody @Valid CreateUserRequest req) {}

    @PutMapping("/{id}")            // PUT /api/users/{id}
    public User update(@PathVariable Long id, @RequestBody UpdateUserRequest req) {}

    @DeleteMapping("/{id}")         // DELETE /api/users/{id}
    public void delete(@PathVariable Long id) {}
}
```

### Extracting Info

- **Context path:** Check `server.servlet.context-path` in application.yml
- **Path prefix:** `@RequestMapping` on class
- **Request body:** `@RequestBody` with DTO class → field types
- **Path vars:** `@PathVariable` → dynamic segments
- **Query params:** `@RequestParam` → query parameters
- **Validation:** `@Valid`, `@NotNull`, `@Size` annotations
- **Security:** `@PreAuthorize`, `SecurityFilterChain` → auth requirements

---

## FastAPI (Python)

### Where to Find Routes

```
app/main.py          — Main app with app.include_router()
app/routers/         — Router modules
app/models/          — Pydantic models
app/dependencies.py  — Shared dependencies
```

### Route Patterns

```python
router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")                    # GET /users
async def list_users(
    skip: int = 0,                  # Query param
    limit: int = 100,               # Query param
):
    pass

@router.get("/{user_id}")          # GET /users/{user_id}
async def get_user(user_id: int):  # Path param (typed)
    pass

@router.post("/", status_code=201) # POST /users
async def create_user(user: UserCreate):  # Body from Pydantic model
    pass

@router.put("/{user_id}")         # PUT /users/{user_id}
async def update_user(user_id: int, user: UserUpdate):
    pass
```

### Extracting Info

- **Router prefix:** `APIRouter(prefix="/prefix")`
- **App mount:** `app.include_router(router, prefix="/api")`
- **Pydantic models:** Define request/response shapes with types
- **Dependencies:** `Depends()` → auth, DB session, etc.
- **Status codes:** `status_code=201` → expected response codes
- **Response model:** `response_model=UserResponse` → response shape

---

## Django REST Framework (Python)

### Where to Find Routes

```
project/urls.py         — Root URL configuration
app/urls.py             — App URL patterns
app/views.py            — View classes/functions
app/serializers.py      — Request/Response serializers
app/permissions.py      — Permission classes
```

### Route Patterns

```python
# urls.py with ViewSet + Router
router = DefaultRouter()
router.register(r'users', UserViewSet)
urlpatterns = [path('api/', include(router.urls))]
# → GET/POST /api/users/
# → GET/PUT/PATCH/DELETE /api/users/{pk}/

# ViewSet
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['get'])
    def me(self, request):    # GET /api/users/me/
        pass

# Function-based views
@api_view(['GET', 'POST'])
@permission_classes([AllowAny])
def user_list(request):
    pass
```

### Extracting Info

- **URL patterns:** `path()`, `re_path()`, `router.register()`
- **Serializers:** Define request/response shapes
- **Permissions:** `permission_classes` → auth requirements
- **ViewSet actions:** Standard CRUD + custom `@action` methods
- **Pagination:** Check `DEFAULT_PAGINATION_CLASS` in settings

---

## Common Analysis Steps

For any framework:

1. **Find the entry point** (main app file)
2. **Trace route registration** (how routes are mounted)
3. **List all endpoints** with methods and paths
4. **Check for global prefix** (e.g., `/api/v1`)
5. **Identify auth middleware** (global vs per-route)
6. **Extract request shapes** from DTOs/models/schemas
7. **Note response codes** for checks
8. **Find validation rules** for generating valid test data
