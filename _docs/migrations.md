# Database Migrations Architecture in Authula

Authula implements a modular, programmatic database migration engine designed specifically for a pluggable authentication framework. Rather than relying on static external SQL migration tools, Authula embeds migrations directly within Go code, separating concerns into **Core Migrations** and **Plugin Migrations** with transactional execution and dependency resolution.

---

## 1. High-Level Architecture Overview

Migrations in Authula are split across two core layers:

```
+--------------------------------------------------------------------+
|                         Caller Interfaces                          |
|  - authula.New() (Auto-migration on startup)                       |
|  - cmd/migrate/main.go (CLI commands: core, plugins, status)       |
|  - Makefile (make migrate-core-up, make migrate-plugins-up)        |
+---------------------------------+----------------------------------+
                                  |
                                  v
+--------------------------------------------------------------------+
|          internal/migrationmanager.Manager                         |
|  - Orchestrates Core and Plugin migrations                         |
|  - Filters plugins via PluginSelector                              |
|  - Resolves dialects (sqlite, postgres, mysql)                     |
+---------------------------------+----------------------------------+
                                  |
                                  v
+--------------------------------------------------------------------+
|               migrations.Migrator (migrations/migrator.go)         |
|  - Manages "auth_schema_migrations" metadata tracking table        |
|  - Resolves plugin dependency DAG (topological sort)               |
|  - Executes Up / Down inside database transactions (bun.Tx)        |
+---------------------------------+----------------------------------+
                                  |
                                  v
+--------------------------------------------------------------------+
|                    Database (PostgreSQL / MySQL / SQLite)          |
+--------------------------------------------------------------------+
```

### Key Components

| File / Component | Purpose |
| :--- | :--- |
| [migrations/migrator.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/migrations/migrator.go) | The low-level migration engine. Handles schema tracking table, transactional execution, and topological sorting of migration dependencies. |
| [internal/migrationmanager/manager.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/internal/migrationmanager/manager.go) | Orchestrator that coordinates running and rolling back Core migrations and Plugin migrations. Supports plugin filtering (`OnlyPluginIDs`, `ExceptPluginIDs`). |
| [migrations/core.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/migrations/core.go) | Programmatic migration definitions for the foundational schema: `users`, `accounts`, `sessions`, and `verifications`. |
| [migrations/provider_variants.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/migrations/provider_variants.go) | Utility (`ForProvider`) to return migrations tailored to specific database providers (`sqlite`, `postgres`, `mysql`). |
| [models/plugin.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/models/plugin.go#L72-L76) | Declares the optional `PluginWithMigrations` interface implemented by plugins with database schemas. |
| [cmd/migrate/main.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/cmd/migrate/main.go) | CLI entry point powered by Cobra for managing migrations independently from application runtime. |

---

## 2. Core vs. Plugin Migrations

Authula clearly demarcates schema ownership into two categories:

### A. Core Migrations (`PluginID = "core"`)
Core migrations establish the fundamental tables required by Authula's identity layer:
- `users`: User identity records (ID, name, email, email_verified, image, metadata, timestamps).
- `accounts`: Linked auth credentials / OAuth providers (user_id, account_id, provider_id, tokens, hashed password).
- `sessions`: Active login sessions (user_id, token, ip_address, user_agent, expires_at).
- `verifications`: One-time verification tokens (user_id, token, type, expires_at) used for email confirmation, OTPs, etc.

These migrations are defined in [migrations/core.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/migrations/core.go) using dialect-specific raw SQL statements executed within transactions:
- `coreSQLiteInitial()` & `coreSQLiteAddMetadata()`
- `corePostgresInitial()` & `corePostgresAddMetadata()`
- `coreMySQLInitial()` & `coreMySQLAddMetadata()`

### B. Plugin Migrations
Plugins can introduce their own tables and indices without mutating core files. Any plugin that requires database persistence implements `models.PluginWithMigrations`:

```go
type PluginWithMigrations interface {
    Migrations(provider string) []migrations.Migration
    DependsOn() []string
}
```

#### Example: Organizations Plugin
In [plugins/organizations/migrationset/migrations.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/plugins/organizations/migrationset/migrations.go):
```go
func ForProvider(provider string) []migrations.Migration {
    return migrations.ForProvider(provider, migrations.ProviderVariants{
        "sqlite":   func() []migrations.Migration { return []migrations.Migration{organizationsSQLiteInitial()} },
        "postgres": func() []migrations.Migration { return []migrations.Migration{organizationsPostgresInitial()} },
        "mysql":    func() []migrations.Migration { return []migrations.Migration{organizationsMySQLInitial()} },
    })
}
```
This migration creates:
- `organizations` table (with foreign key referencing `users(id)`).
- `organization_members` table.
- `organization_invitations` table.

---

## 3. Dependency Resolution & Topological Sorting

A plugin often requires tables from another plugin or the core system. For instance:
- `organizations` plugin depends on `core` (foreign key to `users`).
- An `audit-log` or `rbac` plugin might depend on `organizations`.

In [migrations/migrator.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/migrations/migrator.go#L370-L450), Authula builds a Directed Acyclic Graph (DAG) and executes a topological sort:
1. Validates each `MigrationSet` (ensuring non-empty `PluginID`, valid version format, and matching `Up`/`Down` callbacks).
2. Computes in-degrees and adjacency maps from the `DependsOn` declarations.
3. Applies Kahn's algorithm or depth-first dependency ordering to sequence the migration execution order.
4. If a cycle is detected, migration terminates immediately with an error (e.g. `cyclic dependency detected`).
5. When rolling back (`RollbackAll`), the ordered sets are executed in reverse order.

---

## 4. The Tracking Table & Transactional Safety

Authula tracks applied migrations in the `auth_schema_migrations` table:

```sql
CREATE TABLE IF NOT EXISTS auth_schema_migrations (
    plugin_id VARCHAR(255) NOT NULL,
    version VARCHAR(255) NOT NULL,
    applied_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (plugin_id, version)
);
```

### Migration Execution Flow
For each migration in the sorted list:
1. **Check State**: Queries `auth_schema_migrations` for `plugin_id = ? AND version = ?`.
2. **Skip if Applied**: If already recorded, skips execution.
3. **Execute in Transaction (`m.db.RunInTx`)**:
   ```go
   err := m.db.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
       if err := migration.Up(ctx, tx); err != nil {
           return err
       }
       // Record entry into auth_schema_migrations within the SAME transaction
       _, err := tx.NewInsert().Model(entry).Exec(ctx)
       return err
   })
   ```
4. If any statement inside `migration.Up` fails, the transaction is rolled back, and no record is added to the migrations table, preventing corrupt partial schema states.

---

## 5. Execution Modes

Authula supports both programmatic runtime migrations and standalone CLI execution:

### Mode 1: Automated on Server Startup
When initializing Authula via `authula.New(&authula.AuthConfig{...})`:
1. `migrationManager.RunCore(...)` runs all pending core migrations.
2. Enabled plugins implementing `PluginWithMigrations` are discovered.
3. `pluginRegistry.RunMigrations(ctx)` resolves dependencies and applies pending plugin migrations.

### Mode 2: Dedicated CLI (`cmd/migrate`)
The CLI provides granular administrative control:

```bash
# Manage core migrations
go run ./cmd/migrate core up
go run ./cmd/migrate core down

# Manage plugin migrations
go run ./cmd/migrate plugins up
go run ./cmd/migrate plugins down

# Filter plugin migrations
go run ./cmd/migrate plugins up --only organizations,admin
go run ./cmd/migrate plugins up --except totp

# Inspect status
go run ./cmd/migrate status
go run ./cmd/migrate status --plugin organizations
```

### Mode 3: Makefile Automation
Convenience commands are defined in [Makefile](file:///home/k/Documents/7proj/better-go-auth/others/authula/Makefile#L110-L135):
- `make migrate-up`: Executes `migrate-core-up` followed by `migrate-plugins-up`.
- `make migrate-down`: Safely rolls down plugins first, then rolls down core.
- `make migrate-status`: Prints status of applied migrations across all plugins.

### Guardrails
Authula implements safeguards in [cmd/migrate/main.go](file:///home/k/Documents/7proj/better-go-auth/others/authula/cmd/migrate/main.go):
- **Cannot drop core while plugins exist**: Rolling back `core` while plugin migrations remain active will abort with an error, preventing orphaned foreign key constraints.
- **Core required before plugins**: You cannot apply plugin migrations if `core` has not yet been applied.
