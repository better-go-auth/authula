# Web Frameworks and Adapters Guide

Authula is designed to fit seamlessly into any Go web application, regardless of whether you use standard `net/http`, lightweight routing libraries (Chi), popular frameworks (Gin, Echo), fasthttp engines (Fiber), or schema-driven tools (Huma).

This guide explains:
1. **The Core Architectural Pattern**: `net/http.Handler` compliance.
2. **First-Party Fiber Adapter** (`adapters/fiber`) and why it exists.
3. **Integration Recipes**:
   - [Fiber (v3)](#fiber-v3)
   - [Gin](#gin)
   - [Chi](#chi)
   - [Echo](#echo)
   - [Huma](#huma)
   - [Standard `net/http`](#standard-nethttp)
4. **Authentication Middleware & Context Bridging**: How to inspect authenticated actors across frameworks.

---

## 1. Core Architectural Pattern

Authula internally uses **Chi v5** (`github.com/go-chi/chi/v5`) for its route dispatching, middleware pipelines, and lifecycle hooks (`OnRequest`, `Before`, `After`, `OnResponse`).

Crucially, in [auth.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/auth.go#L264):
```go
func (auth *Auth) Handler() http.Handler
```

By exposing a standard Go `http.Handler`, Authula is **natively compatible with the entire Go HTTP ecosystem**. Any router or framework that implements or adapts `http.Handler` can mount Authula with a single line of code.

---

## 2. The Fiber Adapter (`adapters/fiber`)

While frameworks like Gin, Chi, and Echo are built on Go's standard `net/http`, **Fiber** is built on **fasthttp** (`github.com/valyala/fasthttp`).

### Why a Custom Adapter Was Built
Fiber's standard utility (`fasthttpadaptor`) has a known limitation:
- It streams request bodies via `fasthttp.Request.BodyStream()`.
- Under certain conditions (such as specific HTTP/1.1 chunked encoding or multi-step middleware pipelines), this stream can be read or discarded prematurely, resulting in an empty reader.
- In an authentication system, losing the request body causes sign-in and sign-up POST requests to fail silently.
- Additionally, standard adapters often mismanage multiple `Set-Cookie` response headers.

### How Authula's Fiber Adapter Solves This
In [adapters/fiber/fiber.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/adapters/fiber/fiber.go) and [adapters/fiber/response_writer.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/adapters/fiber/response_writer.go):
1. **Preserves the Body**: Directly extracts the raw bytes (`c.Body()`) and constructs a standard `*http.Request` with `io.NopCloser(bytes.NewReader(body))`.
2. **Propagates Headers**: Copies all incoming HTTP headers, hostnames, and IP addresses into the `net/http` request.
3. **Captures Multi-Value Headers**: A custom `fiberResponseWriter` intercepts `WriteHeader()` and `Header()`, ensuring multiple cookies (e.g., session cookies + CSRF tokens) are properly registered in Fiber's response context.
4. **Executes `handler.ServeHTTP(rw, req)`** transparently.

---

## 3. Integration Recipes

### Fiber (v3)
Use Authula's dedicated adapter in [adapters/fiber](file:///home/k/Documents/7proj/better-go-auth/others/authula/adapters/fiber):

```go
package main

import (
    "log"

    "github.com/gofiber/fiber/v3"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
    fiberadapter "github.com/Authula/authula/adapters/fiber"
)

func main() {
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:3000"),
            authulaconfig.WithBasePath("/api/auth"),
        ),
    })

    app := fiber.New()

    // Mount Authula on /api/auth
    app.Use("/api/auth", fiberadapter.New(fiberadapter.Config{
        Handler: auth.Handler(),
    }))

    app.Get("/health", func(c fiber.Ctx) error {
        return c.SendString("OK")
    })

    log.Fatal(app.Listen(":3000"))
}
```

---

### Gin
Gin provides `gin.WrapH` out of the box to mount any `http.Handler`:

```go
package main

import (
    "github.com/gin-gonic/gin"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
)

func main() {
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:8080"),
            authulaconfig.WithBasePath("/api/auth"),
        ),
    })

    r := gin.Default()

    // Route all /api/auth traffic to Authula
    authHandler := gin.WrapH(auth.Handler())
    r.Any("/api/auth/*any", authHandler)

    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "pong"})
    })

    r.Run(":8080")
}
```

---

### Chi
Because Authula's internal router is also Chi, integration with host Chi routers is 100% native:

```go
package main

import (
    "net/http"

    "github.com/go-chi/chi/v5"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
)

func main() {
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:8080"),
            authulaconfig.WithBasePath("/api/auth"),
        ),
    })

    r := chi.NewRouter()

    // Mount Authula directly as a sub-router
    r.Mount("/api/auth", auth.Handler())

    http.ListenAndServe(":8080", r)
}
```

---

### Echo
Echo provides `echo.WrapHandler`:

```go
package main

import (
    "github.com/labstack/echo/v4"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
)

func main() {
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:8080"),
            authulaconfig.WithBasePath("/api/auth"),
        ),
    })

    e := echo.New()

    // Mount Authula on /api/auth
    e.Any("/api/auth/*", echo.WrapHandler(auth.Handler()))

    e.Start(":8080")
}
```

---

### Huma
[Huma](https://huma.rocks/) is a modern REST/OpenAPI framework for Go. It is built **on top of** an underlying HTTP router (such as Chi, Gin, or standard `http.ServeMux`).

To use Authula with Huma:
1. Initialize the underlying router (e.g. Chi or Gin).
2. Mount `auth.Handler()` on the underlying router.
3. Attach Huma to the same underlying router for your OpenAPI endpoints.

```go
package main

import (
    "net/http"

    "github.com/danielgtaylor/huma/v2"
    "github.com/danielgtaylor/huma/v2/adapters/humachi"
    "github.com/go-chi/chi/v5"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
)

func main() {
    // 1. Create underlying Chi router
    router := chi.NewRouter()

    // 2. Initialize Authula
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:8888"),
            authulaconfig.WithBasePath("/api/auth"),
        ),
    })

    // 3. Mount Authula on the router
    router.Mount("/api/auth", auth.Handler())

    // 4. Attach Huma to the same router
    api := humachi.New(router, huma.DefaultConfig("My Huma API", "1.0.0"))

    // 5. Define your Huma operations
    huma.Register(api, huma.Operation{
        OperationID: "get-profile",
        Method:      http.MethodGet,
        Path:        "/profile",
        Summary:     "Get user profile",
    }, func(ctx huma.Context, input *struct{}) (*struct{ Body string }, error) {
        return &struct{ Body string }{Body: "Hello from Huma!"}, nil
    })

    http.ListenAndServe(":8888", router)
}
```

---

### Standard `net/http`
Standard Go `http.ServeMux` requires zero adapters:

```go
package main

import (
    "net/http"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
)

func main() {
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:8080"),
            authulaconfig.WithBasePath("/api/auth"),
        ),
    })

    mux := http.NewServeMux()

    // Mount directly on standard mux
    mux.Handle("/api/auth/", auth.Handler())

    http.ListenAndServe(":8080", mux)
}
```

---

## 4. Middleware & Actor Extraction

Once Authula authenticates a user or session, downstream application routes need to identify the current actor.

### Context Helpers
Authula provides helper methods in [auth.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/auth.go#L249-L259):
```go
// From context.Context
actor, ok := auth.GetActorFromContext(ctx)

// From *http.Request
actor, ok := auth.GetActorFromRequest(req)
```

The `Actor` contains:
- `ID`: User ID
- `Email`: User Email
- `Session`: Active Session details
- `Metadata`: Custom attributes

### Protected Route Middleware in Gin
```go
func RequireAuth(auth *authula.Auth) gin.HandlerFunc {
    return func(c *gin.Context) {
        actor, ok := auth.GetActorFromRequest(c.Request)
        if !ok {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        c.Set("actor", actor)
        c.Next()
    }
}
```

### Protected Route Middleware in Fiber
```go
func RequireAuth(auth *authula.Auth) fiber.Handler {
    return func(c fiber.Ctx) error {
        // Build lightweight request or inspect token/session cookie
        cookie := c.Cookies("auth_session")
        if cookie == "" {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{"error": "unauthorized"})
        }
        return c.Next()
    }
}
```
