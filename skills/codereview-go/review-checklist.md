# Go Code Review Checklist

Detailed reference for each review category. Agents should use this as a comprehensive guide.

## 1. Correctness & Safety

### Goroutine Lifecycle

```go
// ❌ BAD: goroutine that never exits
func startWorker() {
    go func() {
        for {
            doWork() // no exit condition
        }
    }()
}

// ✅ GOOD: goroutine with controlled lifecycle
func startWorker(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
                doWork()
            }
        }
    }()
}
```

### Context Propagation

```go
// ❌ BAD: context.Background() in request handler
func Handler(w http.ResponseWriter, r *http.Request) {
    result, err := db.QueryContext(context.Background(), query)
}

// ✅ GOOD: use request context
func Handler(w http.ResponseWriter, r *http.Request) {
    result, err := db.QueryContext(r.Context(), query)
}
```

### Defer Pitfalls

```go
// ❌ BAD: defer in loop - resources pile up
for _, f := range files {
    fd, _ := os.Open(f)
    defer fd.Close() // only closes when function returns
}

// ✅ GOOD: use closure or explicit close
for _, f := range files {
    func() {
        fd, _ := os.Open(f)
        defer fd.Close()
        process(fd)
    }()
}
```

### Nil Safety

```go
// ❌ BAD: no nil check before dereference
resp, err := http.Get(url)
defer resp.Body.Close() // panics if resp is nil
if err != nil {
    return err
}

// ✅ GOOD: check error first
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

### Type Assertion Safety

```go
// ❌ BAD: unchecked type assertion
val := m["key"].(string) // panics if wrong type or nil

// ✅ GOOD: checked assertion
val, ok := m["key"].(string)
if !ok {
    return fmt.Errorf("unexpected type for key")
}
```

---

## 2. Concurrency & Performance

### Data Race Detection

```go
// ❌ BAD: unsynchronized shared state
var count int
for i := 0; i < 10; i++ {
    go func() {
        count++ // data race
    }()
}

// ✅ GOOD: use atomic or mutex
var count atomic.Int64
for i := 0; i < 10; i++ {
    go func() {
        count.Add(1)
    }()
}
```

### Channel Safety

```go
// ❌ BAD: goroutine leaks on unbuffered channel
func fetch(urls []string) []string {
    ch := make(chan string)
    for _, url := range urls {
        go func(u string) {
            ch <- http.Get(u) // blocks forever if caller returns early
        }(url)
    }
    return <-ch // only reads one, rest leak
}

// ✅ GOOD: buffered channel + context cancellation
func fetch(ctx context.Context, urls []string) []string {
    ch := make(chan string, len(urls))
    for _, url := range urls {
        go func(u string) {
            select {
            case ch <- doFetch(u):
            case <-ctx.Done():
                return
            }
        }(url)
    }
    // collect results...
}
```

### Unbounded Goroutine Creation

```go
// ❌ BAD: unbounded goroutines in request handler
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    go processAsync(r) // one goroutine per request, no limit
}

// ✅ GOOD: worker pool or semaphore
var sem = make(chan struct{}, 100) // limit to 100 concurrent
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    sem <- struct{}{}
    go func() {
        defer func() { <-sem }()
        processAsync(r)
    }()
}
```

---

## 3. Error Handling

### Silent Failure

```go
// ❌ BAD: error silently discarded
json.Unmarshal(data, &config) // error ignored
db.Close()                     // error ignored

// ✅ GOOD: handle or explicitly mark
if err := json.Unmarshal(data, &config); err != nil {
    return fmt.Errorf("parse config: %w", err)
}
```

### Error Context Preservation

```go
// ❌ BAD: context lost
if err != nil {
    return err // no context about what operation failed
}

// ❌ BAD: wraps with %v, breaks error chain
return fmt.Errorf("query failed: %v", err)

// ✅ GOOD: wraps with %w, preserves chain
return fmt.Errorf("query user %d: %w", userID, err)
```

### Panic as Control Flow

```go
// ❌ BAD: panic for expected conditions
func MustGetConfig(key string) string {
    val, ok := config[key]
    if !ok {
        panic("missing config: " + key) // should return error
    }
    return val
}

// ✅ GOOD: return error for expected failures
func GetConfig(key string) (string, error) {
    val, ok := config[key]
    if !ok {
        return "", fmt.Errorf("missing config key: %s", key)
    }
    return val, nil
}
```

---

## 4. Security

### SQL Injection

```go
// ❌ BAD: string concatenation in SQL
query := "SELECT * FROM users WHERE name = '" + name + "'"
db.Query(query)

// ✅ GOOD: parameterized query
db.Query("SELECT * FROM users WHERE name = ?", name)
```

### Command Injection

```go
// ❌ BAD: user input in exec
cmd := exec.Command("sh", "-c", "echo " + userInput)

// ✅ GOOD: separate command and args
cmd := exec.Command("echo", userInput)
```

### Path Traversal

```go
// ❌ BAD: user input in file path
filePath := filepath.Join(baseDir, userInput)
data, _ := os.ReadFile(filePath) // "../../../etc/passwd"

// ✅ GOOD: validate after join
filePath := filepath.Join(baseDir, filepath.Clean(userInput))
if !strings.HasPrefix(filePath, baseDir) {
    return fmt.Errorf("invalid path")
}
```

### Insecure Randomness

```go
// ❌ BAD: math/rand for security tokens
token := fmt.Sprintf("%x", rand.Int63())

// ✅ GOOD: crypto/rand for security
b := make([]byte, 32)
crypto_rand.Read(b)
token := hex.EncodeToString(b)
```

### Hardcoded Secrets

```go
// ❌ BAD: hardcoded credentials
const apiKey = "sk-1234567890abcdef"
db, _ := sql.Open("postgres", "user:password@localhost/db")

// ✅ GOOD: environment variables or secret manager
apiKey := os.Getenv("API_KEY")
dsn := os.Getenv("DATABASE_URL")
```

---

## 5. Observability

### Structured Logging

```go
// ❌ BAD: unstructured logging
log.Printf("user %s logged in from %s", user, ip)

// ✅ GOOD: structured logging with slog
slog.Info("user logged in",
    slog.String("user", user),
    slog.String("ip", ip),
    slog.String("request_id", requestID),
)
```

### Sensitive Data in Logs

```go
// ❌ BAD: logging sensitive data
slog.Info("auth attempt", "password", password, "token", token)

// ✅ GOOD: redact sensitive fields
slog.Info("auth attempt", "user", email, "token_prefix", token[:8]+"...")
```

---

## 6. Architecture & API Design (Optimize Only)

### Interface Consumer Rule

```go
// ❌ SUBOPTIMAL: interface defined by implementor
// package user
type UserService interface { // defined where implemented
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
}

// ✅ BETTER: interface defined by consumer
// package handler
type UserGetter interface { // defined where used
    GetUser(id int) (*User, error)
}
```

### Global State

```go
// ❌ SUBOPTIMAL: package-level mutable state
var db *sql.DB
func init() {
    db, _ = sql.Open("postgres", os.Getenv("DSN"))
}

// ✅ BETTER: dependency injection
type Server struct {
    db *sql.DB
}
func NewServer(db *sql.DB) *Server {
    return &Server{db: db}
}
```
