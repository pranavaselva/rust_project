# rust_project

Architecture Overview
This project utilizes a decoupled, three-tier architecture designed for high performance and strict type safety.

Frontend (Angular): The user interface is built with Angular components. These components capture user events (like form submissions or button clicks) and delegate data handling to Angular Services. These services use the HttpClient module to communicate with the backend via RESTful APIs.

Backend (Rust): The server is powered by Rust (using the Axum/Actix framework). Rust provides a high-performance, memory-safe environment to process business logic. It receives HTTP requests, validates the data, and interacts with the database.

Database (PostgreSQL): All persistent data is stored in PostgreSQL. Rust communicates with the database using a driver or ORM (like sqlx or Diesel), ensuring that data remains consistent and structured.

Reflection: Why Separation Matters
Separating the frontend and backend is crucial for scalability and maintainability. By decoupling the two, the backend can be optimized for heavy processing without affecting the user experience, while the frontend can be updated or redesigned independently. Additionally, the combination of TypeScript on the frontend and Rust on the backend provides end-to-end type safety, significantly reducing runtime errors.

(![Flow Diagram](<rust image.png>))

---

## Detailed API Explanation: Route → Handler → Logic → Response

### 1. How Rust Routes Map to the Correct Handler

In Axum (the async web framework used in this project), routes are defined as a tower service that maps HTTP paths to handler functions. When a request arrives at the server:

```rust
// Routes are defined at startup
let app = Router::new()
    .route("/api/users/:id", get(get_user_handler))
    .route("/api/users", post(create_user_handler))
    .route("/api/users/:id", put(update_user_handler));
```

The router examines the incoming request's HTTP method (GET, POST, PUT, etc.) and path, then dispatches it to the corresponding handler function. This is **compile-time verified** — if a route doesn't have a matching handler, the code won't compile. Path parameters like `:id` are automatically extracted and passed to the handler.

### 2. How the Handler Receives and Processes the Request

When a request reaches a handler, Rust's extractors automatically parse and validate the incoming data. Extractors are middleware-like components that transform raw HTTP data into Rust types:

```rust
async fn create_user_handler(
    State(db): State<PgPool>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<UserResponse>, AppError> {
    // `State(db)` extracts the database connection pool
    // `Json(payload)` deserializes the JSON body into CreateUserRequest struct

    // At this point, we KNOW:
    // - The JSON was valid and well-formed
    // - The data conforms to the CreateUserRequest schema
    // - All required fields are present and correctly typed
    // - Optional fields are wrapped in Option<T>
    // - All validation decorators (email, length, etc.) have passed

    // If ANY of the above fails, the request never reaches this code.
    // Instead, a 400 BadRequest is sent automatically with validation errors.

    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *"
    )
    .bind(&payload.name)
    .bind(&payload.email)
    .fetch_one(&db)
    .await?;

    Ok(Json(UserResponse::from(user)))
}

// Example: Incoming request from Angular
// POST /api/users
// Content-Type: application/json
//
// REQUEST BODY (Valid):
// {"name": "Alice", "email": "alice@example.com"}
// ✓ Deserializes successfully
// ✓ Handler executes
//
// REQUEST BODY (Invalid - missing email):
// {"name": "Bob"}
// ✗ Deserialization fails
// ✗ 400 BadRequest returned: {"error": "missing field 'email'"}
// ✗ Handler never executes
```

**Key Points:**

- The handler function signature IS the API contract — no separate schema file needed
- Invalid requests are rejected by the extraction layer before handler code runs
- Type mismatches are caught at compile time during development, not at runtime in production
- This prevents the "garbage in, garbage out" problem common in dynamically-typed backends

### 3. How Typed Structs Ensure Safe Input/Output

All data structures are defined as strongly-typed Rust structs. This ensures compile-time safety:

```rust
// Input validation through the type system
#[derive(Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 1, max = 255))]
    pub name: String,

    #[validate(email)]
    pub email: String,
}

// Output serialization through the type system
#[derive(Serialize, Sqlx)]
pub struct UserResponse {
    pub id: i32,
    pub name: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
}
```

**Safety Guarantees:**

- **Deserialization**: If the JSON doesn't match the struct fields, deserialization fails before the handler executes
- **Serialization**: The response is guaranteed to match the declared types — no null pointer exceptions or type coercion surprises
- **Database Mapping**: `#[derive(Sqlx)]` ensures that database columns exactly match struct fields. A column rename causes a compile error, not a silent null value
- **Compile-Time Verification**: The Rust compiler verifies that every field used in business logic matches the struct definition. Typos cause build failures, not runtime panics.

**Example of Type Safety in Action:**

```rust
// ✓ This compiles: all fields are present and correctly typed
let response = UserResponse {
    id: user.id,
    name: user.name,
    email: user.email,
    created_at: user.created_at,
};

// ✗ This WON'T compile: missing the 'email' field
let response = UserResponse {
    id: user.id,
    name: user.name,
    // ^^^ COMPILER ERROR: missing field 'email'
    created_at: user.created_at,
};

// ✗ This WON'T compile: wrong type for 'id' (String instead of i32)
let response = UserResponse {
    id: "wrong".to_string(),  // ^^^ TYPE ERROR: expected i32, found String
    name: user.name,
    email: user.email,
    created_at: user.created_at,
};
```

### 4. How Your API Returns JSON Responses

Axum automatically serializes Rust types to JSON responses:

```rust
#[derive(Serialize)]
pub struct ApiResponse<T> {
    pub success: bool,
    pub data: Option<T>,
    pub error: Option<String>,
}

// Handler returns the typed response
async fn get_user_handler(
    State(db): State<PgPool>,
    Path(id): Path<i32>,
) -> Result<Json<ApiResponse<UserResponse>>, AppError> {
    let user = sqlx::query_as::<_, User>(
        "SELECT * FROM users WHERE id = $1"
    )
    .bind(id)
    .fetch_optional(&db)
    .await?;

    match user {
        Some(u) => Ok(Json(ApiResponse {
            success: true,
            data: Some(UserResponse::from(u)),
            error: None,
        })),
        None => Ok(Json(ApiResponse {
            success: false,
            data: None,
            error: Some("User not found".to_string()),
        })),
    }
}

// Example HTTP Responses:
//
// SUCCESS (HTTP 200 OK):
// {
//   "success": true,
//   "data": {
//     "id": 1,
//     "name": "Alice",
//     "email": "alice@example.com",
//     "created_at": "2024-01-15T10:30:00Z"
//   },
//   "error": null
// }
//
// NOT FOUND (HTTP 200 OK, but success=false):
// {
//   "success": false,
//   "data": null,
//   "error": "User not found"
// }
//
// VALIDATION ERROR (HTTP 400 Bad Request):
// {
//   "success": false,
//   "data": null,
//   "error": "invalid email format"
// }
```

The `Json` wrapper tells Axum to serialize the response with `application/json` content type. The response is automatically validated against the type signature. **Importantly**, if you try to return a field that doesn't exist in the response struct, the code won't compile—type safety extends to the response layer.

### 5. How SQLx/SeaORM Interacts with PostgreSQL

SQLx is a compile-time checked SQL query library. This is where Rust shines:

```rust
// Compile-time verification: SQLx checks this query against your live database schema
let user = sqlx::query_as::<_, User>(
    "SELECT id, name, email, created_at FROM users WHERE id = $1"
)
.bind(id)  // $1 is type-checked: must match id's type
.fetch_one(&db)
.await?;

// The User struct maps EXACTLY to the SELECT columns:
#[derive(Sqlx)]
pub struct User {
    pub id: i32,           // ← Column 'id' INT must match
    pub name: String,      // ← Column 'name' VARCHAR must match
    pub email: String,     // ← Column 'email' VARCHAR must match
    pub created_at: DateTime<Utc>, // ← Column 'created_at' TIMESTAMP must match
}

// If the database schema changes, SQLx recompilation will FAIL with clear errors:
// ERROR: column "created_at" does not exist
// ERROR: column "name" has type character varying, but struct field expects i32
// If you mistype a column name: SELECT id, names FROM users -- FAILS at compile time
// If you return wrong columns: SELECT id, name FROM users (missing created_at) -- FAILS
// If you map to wrong struct: query_as::<_, WrongStruct> -- TYPE MISMATCH ERROR
```

**Key Differences from Runtime-Checked ORMs:**

| Aspect              | SQLx (Rust)                | Traditional ORMs (Python/Node.js) |
| ------------------- | -------------------------- | --------------------------------- |
| **Query Checking**  | Compile-time (build fails) | Runtime (crashes in production)   |
| **Schema Mismatch** | Compile error              | Silent null or type coercion      |
| **Column Rename**   | Compiler forces update     | Deploy breaks silently            |
| **Type Safety**     | Guaranteed at compile time | Runtime type errors possible      |
| **Discovery**       | Found during development   | Found by customers in production  |

**Why This Matters:**

- In Python/Node ORMs, a database migration might add a column, and old code continues to work with `null` values silently
- In Rust/SQLx, the same migration causes a compile error that blocks deployment
- You can't ship broken code—the compiler won't let you

---

## End-to-End API Request Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Angular UI Component (TypeScript)                           │
│     └─> User clicks button or submits form                      │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Angular Service (HttpClient)                                │
│     └─> this.http.post('/api/users', userData)                  │
│     └─> Observable<UserResponse> returned                       │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTP POST /api/users
                       │ Content-Type: application/json
                       │ Body: {"name": "Alice", "email": "..."}
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Rust Server Routing (Axum)                                  │
│     └─> POST /api/users → create_user_handler                   │
│     └─> Router matches method & path at compile-time            │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Extraction & Validation Layer                               │
│     └─> Json(payload) deserializes JSON body                    │
│     └─> Validates: field types, string length, email format     │
│     └─> Returns 400 BadRequest if validation fails              │
│     └─> State(db) provides database connection pool             │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Business Logic (Handler Function)                           │
│     └─> Input: CreateUserRequest (strongly typed)               │
│     └─> Perform domain logic (check duplicates, etc.)           │
│     └─> Call SQLx query builder                                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. SQLx Query (Compile-Time Verified)                          │
│     └─> INSERT INTO users (name, email) VALUES ($1, $2)         │
│     └─> Column names & types verified at compile time           │
│     └─> Parameter binding: $1 = name, $2 = email               │
└──────────────────────┬──────────────────────────────────────────┘
                       │ PostgreSQL Protocol
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. PostgreSQL Database                                         │
│     └─> Execute INSERT statement                                │
│     └─> Enforce schema constraints                              │
│     └─> Return inserted row                                     │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. Row Mapping (SQLx)                                          │
│     └─> Database row → User struct (column-by-column)           │
│     └─> Type mismatch = compile error, not silent null          │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  9. Serialization (serde_json + Axum)                           │
│     └─> User struct → UserResponse struct                       │
│     └─> UserResponse → JSON string                              │
│     └─> Set Content-Type: application/json                      │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTP 201 Created
                       │ {"id": 42, "name": "Alice", ...}
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  10. Angular Service (Observable Resolution)                    │
│      └─> HTTP response received (201 Created)                   │
│      └─> JSON parsed as UserResponse (TypeScript typed)         │
│      └─> Observable emits the typed data                        │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  11. Angular Component (Subscribe Handler)                      │
│      └─> Receives typed UserResponse                            │
│      └─> Update component state (typed)                         │
│      └─> Angular change detection triggers                      │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  12. Angular Template (Rendering)                               │
│      └─> Display new user in table/list                         │
│      └─> Update UI with confirmation message                    │
│      └─> Request complete, user sees result                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why Type-Safe, Strongly-Validated Rust APIs Matter in Modern Backend Development

**Type safety at the API boundary is not just a nice-to-have—it's a fundamental requirement for building reliable production systems.** Here's why:

1. **Compile-Time Verification**: Unlike dynamically-typed languages where API contract violations are discovered in production (after customers experience outages), Rust enforces API contracts at compile time. If a database schema changes or a request struct field is renamed, the build fails immediately during development, not in production.

2. **Memory Safety Without Garbage Collection**: Rust's ownership system eliminates entire classes of bugs (null pointer dereferences, use-after-free, data races) that plague other systems languages. Your API handlers are guaranteed memory-safe, meaning you won't face mysterious crashes or memory leaks after days of uptime.

3. **Performance Predictability**: Rust's zero-cost abstractions and tight control over resource allocation mean your API responds consistently under load. No garbage collection pauses, no unexpected thread spikes—just predictable, millisecond-level latency that scales linearly with the request rate.

4. **Data Consistency Guarantees**: The combination of typed structs and compile-time SQL verification ensures that data flowing through your system matches its declared schema. There's no possibility of partial nulls, type coercion surprises, or impedance mismatches between your code and database. What you write is what you get.

5. **Fearless Refactoring**: In a strongly-typed language like Rust, refactoring is safe. Rename a field, change a type, modify a database column—the compiler immediately shows every place that needs updating. In contrast, dynamic languages require manual testing or runtime errors to catch these issues.

6. **Production Reliability**: The combination of these properties means that Rust APIs exhibit exceptional stability. Security vulnerabilities are fewer (no buffer overflows, no use-after-frees), performance is predictable, and bugs that do slip through are typically logic errors (caught by testing) rather than system-level crashes.

By investing in type safety and strong validation at the API layer, you're building a foundation that scales with your application's complexity, reduces on-call incidents, and minimizes expensive debugging sessions in production.
