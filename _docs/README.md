# Authula Architecture & Integration Documentation

This directory contains in-depth documentation detailing how **Authula** handles database migrations, its database/ORM design, and its framework adapter architecture (including support for Fiber, Gin, Chi, Huma, GORM, Bun, SQLC, etc.).

## Table of Contents

1. [Database Migrations Architecture](file:///home/k/Documents/7proj/better-go-auth/others/authula/_docs/migrations.md)
   - Comprehensive guide on how migrations work in Authula.
   - Low-level `Migrator` engine vs. high-level `Manager`.
   - Core migrations vs. Plugin migrations (`PluginWithMigrations`).
   - Directed Acyclic Graph (DAG) dependency resolution and topological sorting.
   - Multi-dialect support (PostgreSQL, MySQL, SQLite).
   - Automatic execution on startup vs. CLI / Makefile commands.
   - Transactional safety and rollback guards.

2. [ORM & Database Abstraction Analysis](file:///home/k/Documents/7proj/better-go-auth/others/authula/_docs/orm-support.md)
   - Detailed breakdown answering whether Authula supports multiple ORMs (GORM, SQLC, Ent, Bun).
   - Internal dependency on Bun ORM (`bun.IDB`, `models.*`, `bun.BaseModel`, `bun_*.go`).
   - Database engines supported (PostgreSQL, MySQL, SQLite).
   - Architectural boundary: Host Application ORM vs. Authula Internal ORM.
   - How to integrate Authula alongside GORM, SQLC, or Ent via shared `*sql.DB` connection pools.
   - Feasibility and design pattern for swapping/extending repository backends.

3. [Web Frameworks & Adapters Guide](file:///home/k/Documents/7proj/better-go-auth/others/authula/_docs/adapters-and-frameworks.md)
   - Analysis of web framework support (Fiber, Gin, Chi, Echo, Huma, standard `net/http`).
   - The foundation: Authula as a standard `net/http.Handler` powered internally by Chi v5.
   - Dedicated first-party adapter for Fiber (`adapters/fiber`) and why `fasthttpadaptor` is bypassed.
   - Step-by-step integration recipes and code snippets for Fiber, Gin, Chi, Echo, and Huma.
   - Middleware propagation, Actor extraction, and context bridging.
