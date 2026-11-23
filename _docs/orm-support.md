# ORM & Database Abstraction in Authula

A common question when adopting Authula is whether it supports multiple ORMs (such as GORM, SQLC, Bun, Ent, etc.) and what database engines it can run against.

This document breaks down:
1. **The Direct Answer**: Does Authula support multiple ORMs?
2. **Authula's Internal ORM Architecture** (Bun ORM).
3. **Database Engine / Dialect Support** (PostgreSQL, MySQL, SQLite).
4. **How Host Applications Use Other ORMs (GORM, SQLC, Ent)** alongside Authula.
5. **Architectural Analysis**: Can Authula's internal ORM be replaced or decoupled?

---

## 1. Quick Summary & Verdict

| Question | Answer | Details |
| :--- | :--- | :--- |
| **Does Authula internally support multiple ORMs?** | **No** | Authula's internal data layer, migrations, models, and repositories are exclusively built on **Bun ORM** (`github.com/uptrace/bun`). |
| **Does Authula support multiple database engines?** | **Yes** | Authula supports **PostgreSQL**, **MySQL**, and **SQLite** via Bun dialects and standard Go SQL drivers. |
| **Can your application use GORM, SQLC, or Ent with Authula?** | **Yes** | Your application can use whatever ORM it wants for business entities. Authula only manages its own tables (`users`, `accounts`, `sessions`, etc.) and can share the underlying standard `*sql.DB` connection pool with your ORM. |

---

## 2. Authula's Internal Dependency on Bun ORM

Authula is strongly and deliberately coupled to **Bun ORM** across all layers of its persistence tier:

### A. Core Models embed `bun.BaseModel`
All internal domain models declare Bun table and column annotations. For example, in [models/user.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/models/user.go):
```go
package models

import (
    "time"
    "github.com/uptrace/bun"
)

type User struct {
    bun.BaseModel `bun:"table:users"`

    ID            string         `json:"id" bun:"column:id,pk"`
    Name          string         `json:"name" bun:"column:name"`
    Email         string         `json:"email" bun:"column:email"`
    EmailVerified bool           `json:"email_verified" bun:"column:email_verified"`
    Image         *string        `json:"image" bun:"column:image"`
    Metadata      map[string]any `json:"metadata" bun:"column:metadata"`
    CreatedAt     time.Time      `json:"created_at" bun:"column:created_at,default:current_timestamp"`
    UpdatedAt     time.Time      `json:"updated_at" bun:"column:updated_at,default:current_timestamp"`
}
```

### B. Repository Interfaces and Transaction Semantics
While Authula promotes repository interfaces (e.g. `UserRepository`, `AccountRepository`, `SessionRepository`), the transaction methods directly reference Bun's `bun.IDB` interface in [core/repositories/interfaces.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/core/repositories/interfaces.go#L36):
```go
type AccountRepository interface {
    GetByID(ctx context.Context, id string) (*models.Account, error)
    // ...
    WithTx(tx bun.IDB) AccountRepository
}
```

### C. Repository Implementations
All built-in repositories in `core/repositories/` and plugin folders (such as `organizations`, `admin`, `api-key`) rely directly on Bun query builders:
- `bun_user_repository.go`
- `bun_account_repository.go`
- `bun_session_repository.go`
- `bun_verification_repository.go`

Example query pattern from [core/repositories/bun_user_repository.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/core/repositories/bun_user_repository.go):
```go
func (r *bunUserRepository) GetByEmail(ctx context.Context, email string) (*models.User, error) {
    user := new(models.User)
    err := r.db.NewSelect().
        Model(user).
        Where("email = ?", email).
        Scan(ctx)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, nil
        }
        return nil, err
    }
    return user, nil
}
```

### D. Migration Engine
As detailed in the Migrations guide, `Migrator` requires a `*bun.DB` and passes `bun.Tx` to all migration callbacks:
```go
type Migration struct {
    Version string
    Up      func(ctx context.Context, tx bun.Tx) error
    Down    func(ctx context.Context, tx bun.Tx) error
}
```

---

## 3. Supported Database Engines

Even though Authula uses Bun exclusively, it supports three major SQL dialects via Bun's dialect drivers in [internal/bootstrap/database.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/internal/bootstrap/database.go):

| Provider | Go Driver | Bun Dialect | Connection Example |
| :--- | :--- | :--- | :--- |
| **PostgreSQL** | `github.com/lib/pq` | `pgdialect.New()` | `postgres://user:pass@localhost:5432/dbname?sslmode=disable` |
| **MySQL** | `github.com/go-sql-driver/mysql` | `mysqldialect.New()` | `user:pass@tcp(localhost:3306)/dbname?parseTime=true` |
| **SQLite** | `github.com/mattn/go-sqlite3` | `sqlitedialect.New()` | `authula.db` or `:memory:` |

Connection pools (`MaxOpenConns`, `MaxIdleConns`, `ConnMaxLifetime`) and debug logging (`bundebug`) are managed centrally during bootstrap.

---

## 4. Using Authula in Apps with GORM, SQLC, or Ent

Because Authula runs as an independent library or standalone service, **your host application does not need to use Bun for its own business domain**.

All major Go ORMs (GORM, SQLC, Ent) operate on top of standard Go `database/sql.DB`. You can easily share the same database connection pool between your ORM and Authula.

### Pattern: Shared Connection Pool

```
                                +-----------------------------+
                                |  Underlying *sql.DB Pool    |
                                +--------------+--------------+
                                               |
                       +-----------------------+-----------------------+
                       |                                               |
                       v                                               v
        +-----------------------------+                 +-----------------------------+
        |       Authula (Bun DB)      |                 |   Your App (GORM / SQLC)    |
        | - users                     |                 | - products                  |
        | - accounts                  |                 | - orders                    |
        | - sessions                  |                 | - invoices                  |
        | - verifications             |                 +-----------------------------+
        +-----------------------------+
```

### Code Example: Sharing Database Connection with GORM

```go
package main

import (
    "log"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "github.com/uptrace/bun"
    "github.com/uptrace/bun/dialect/pgdialect"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
)

func main() {
    dsn := "postgres://user:pass@localhost:5432/myapp?sslmode=disable"

    // 1. Initialize your application's GORM database
    gormDB, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }

    // 2. Extract standard *sql.DB pool from GORM
    sqlDB, err := gormDB.DB()
    if err != nil {
        log.Fatal(err)
    }

    // 3. Wrap the existing *sql.DB pool into a Bun instance for Authula
    bunDB := bun.NewDB(sqlDB, pgdialect.New())

    // 4. Pass bunDB directly to Authula
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithBaseURL("http://localhost:8080"),
            authulaconfig.WithBasePath("/api/auth"),
            authulaconfig.WithDatabaseProvider("postgres"),
        ),
        DB: bunDB, // Authula uses your existing database connection!
    })

    log.Println("Authula initialized with shared GORM connection pool")
}
```

### Code Example: Sharing Database Connection with SQLC

With SQLC, you generate type-safe Go code from SQL queries against a standard `*sql.DB` or `*sql.Tx`:

```go
package main

import (
    "database/sql"
    _ "github.com/lib/pq"
    "github.com/uptrace/bun"
    "github.com/uptrace/bun/dialect/pgdialect"

    authula "github.com/Authula/authula"
    authulaconfig "github.com/Authula/authula/config"
    "myapp/db/generated_sqlc"
)

func main() {
    // 1. Open standard *sql.DB
    sqlDB, _ := sql.Open("postgres", "postgres://user:pass@localhost:5432/myapp?sslmode=disable")

    // 2. SQLC queries use the sqlDB directly
    queries := generated_sqlc.New(sqlDB)

    // 3. Wrap sqlDB into Bun for Authula
    bunDB := bun.NewDB(sqlDB, pgdialect.New())

    // 4. Initialize Authula
    auth := authula.New(&authula.AuthConfig{
        Config: authulaconfig.NewConfig(
            authulaconfig.WithDatabaseProvider("postgres"),
        ),
        DB: bunDB,
    })

    _ = queries
    _ = auth
}
```

---

## 5. Architectural Analysis: Why Bun & What Would Multi-ORM Require?

### Why Authula Selected Bun
1. **Lightweight & High Performance**: Unlike GORM, Bun is not reflection-heavy; it compiles queries fast and has minimal overhead.
2. **Explicit SQL**: Bun does not try to hide SQL behind massive layers of abstraction; it generates clean SQL matching Authula's multi-dialect requirements (`mysqldialect`, `pgdialect`, `sqlitedialect`).
3. **Built-in Transaction Runner**: `RunInTx` with context propagation makes atomic multi-operation auth logic simple.

### What Would Be Required to Support Native Swappable ORMs?
If someone wanted to replace Bun with GORM, SQLC, or Ent *internally* in Authula, the following changes would be needed:
1. **Abstract `bun.IDB` and `bun.Tx`**: Repositories currently have `WithTx(tx bun.IDB)`. This would need to be abstracted to a generic transaction handle or context-based transaction manager.
2. **Decouple Struct Tags**: Models like `User` and `Account` would need either separate ORM entities or dual tags (`gorm:"column:id;primaryKey"` alongside `bun:"..."`).
3. **Rewrite Repository Implementations**: Implement `GormUserRepository`, `SqlcUserRepository`, etc., for core and all plugins.
4. **Decouple the Migration Engine**: The `Migrator` in `migrations/migrator.go` uses `tx.NewInsert()`, `m.db.NewCreateTable()`, and `tx bun.Tx`. It would need to accept raw `*sql.DB` or `*sql.Tx` rather than `bun.DB`.

**Conclusion**: Authula deliberately chose Bun as its internal engine to maintain consistency and simplicity across plugins, while allowing developers to easily plug Authula into any application using other ORMs via shared `*sql.DB` connections.
