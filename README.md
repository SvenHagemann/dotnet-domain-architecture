# .NET Domain Architecture

> A pragmatic reference architecture for **ASP.NET Core, Domain-Driven Design, Entity Framework Core and PostgreSQL** — with **Single-Tenancy as the simple default** and **Multi-Tenancy as an optional capability**.

[![.NET](https://img.shields.io/badge/.NET-10-512BD4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![ASP.NET Core](https://img.shields.io/badge/ASP.NET%20Core-10-512BD4?logo=dotnet&logoColor=white)](https://learn.microsoft.com/aspnet/core/)
[![Entity Framework Core](https://img.shields.io/badge/EF%20Core-10-512BD4?logo=dotnet&logoColor=white)](https://learn.microsoft.com/ef/core/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![License](https://img.shields.io/badge/license-see%20LICENSE-blue.svg)](LICENSE)

---

## Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Core Principles](#core-principles)
- [Optional Capabilities](#optional-capabilities)
- [Domain Model](#domain-model)
- [Persistence](#persistence)
- [Reference Data](#reference-data)
- [Caching](#caching)
- [Multi-Tenancy](#multi-tenancy)
- [CQRS & Queries](#cqrs--queries)
- [Repository Structure](#repository-structure)
- [Architecture Prompt](#architecture-prompt)
- [Status](#status)
- [License](#license)

---

## Overview

This repository explores a **production-oriented .NET architecture** for long-lived business applications.

The focus is not on collecting patterns, but on defining **clear architectural boundaries** and using abstractions only where they provide concrete value.

The architecture addresses:

- Domain-Driven Design
- Clean Architecture
- Aggregate-oriented persistence
- Strongly Typed IDs
- EF Core + Npgsql + PostgreSQL
- Reference Data (fachliches Datenkonzept)
- optional Caching
- optional Auditing
- optional Multi-Tenancy
- pragmatic CQRS
- testable persistence
- concurrency and security
- horizontal scaling

The primary example is an **Ordering / Order domain**.

---

## Architecture

The dependency direction is deliberately simple:

```text
┌─────────────────────────────────────┐
│               Domain                │
│                                     │
│ Entities · Aggregates · Value       │
│ Objects · Domain Rules              │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│            Application              │
│                                     │
│ Commands · Queries · Use Cases      │
│ Repository / Domain Abstractions    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│          Infrastructure             │
│                                     │
│ EF Core · Npgsql · PostgreSQL       │
│ Repositories · Cache · Interceptors │
└─────────────────────────────────────┘
```

The Domain has no dependency on:

```text
EF Core
Npgsql
PostgreSQL
Redis
ASP.NET Core
HttpContext
IMemoryCache
IDistributedCache
```

Infrastructure implements the abstractions required by the inner layers.

---

## Core Principles

### 1. Aggregates are the persistence boundary

Repositories operate on **Aggregate Roots**, not on every entity.

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

`OrderLine` and `ShippingAddress` therefore do not receive independent repositories.

---

### 2. Keep the base architecture small

Not every class needs an interface.

Not every database table needs a repository.

Not every query needs CQRS.

Not every application needs Multi-Tenancy.

The architecture prefers:

> **The simplest abstraction that correctly expresses the business or technical requirement.**

---

### 3. Strongly Typed IDs

Identifiers are modeled explicitly:

```csharp
public readonly record struct OrderId(Guid Value);
public readonly record struct OrderLineId(Guid Value);
public readonly record struct TenantId(Guid Value);
public readonly record struct UserId(Guid Value);
public readonly record struct OrderStatusId(int Value);
```

This prevents accidental mixing of unrelated identifiers at compile time.

---

## Optional Capabilities

The base architecture is independent of cross-cutting capabilities.

```text
                    BASE ARCHITECTURE
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
        Audit        Multi-Tenancy       Caching
       optional         optional         optional
```

### Audit

Auditing is preferably implemented through composition and infrastructure mechanisms such as:

```text
IAuditableEntity
     +
SaveChangesInterceptor
     +
ICurrentUser
     +
TimeProvider
```

No inheritance hierarchy such as `AuditableTenantAggregateRoot` is required.

### Multi-Tenancy

Multi-Tenancy is explicitly opt-in.

Single-Tenant applications do **not** require:

```text
ITenantContext
TenantId
Tenant Repository
Tenant Query Filters
Tenant Resolution
Tenant-aware Cache Keys
```

unless the application actually needs them.

### Caching

Caching is a performance layer:

```text
PostgreSQL
    =
Source of Truth

Memory / Redis
    =
Performance Layer
```

Caching must never become the authoritative persistence mechanism.

---

## Domain Model

The primary example is the Ordering domain:

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

with Strongly Typed IDs:

```text
OrderId
OrderLineId
TenantId
UserId
OrderStatusId
```

The model demonstrates:

- Aggregate Root boundaries
- Child Entity lifecycle
- encapsulation of collections
- Value Objects
- optional tenant awareness
- optional auditing
- domain identity
- persistence mapping

Preferred aggregate interaction:

```csharp
order.AddLine(...);
order.RemoveLine(...);
order.ChangeShippingAddress(...);
```

rather than exposing mutable child collections.

---

## Persistence

**PostgreSQL is the concrete persistence provider and system of record.**

```text
Domain
   │
   ▼
Application
   │
   ▼
Repository / Query
   │
   ▼
EF Core
   │
   ▼
Npgsql
   │
   ▼
PostgreSQL
```

The architecture explicitly evaluates PostgreSQL-specific capabilities such as:

- `uuid`
- `integer` / `bigint`
- foreign keys
- unique constraints
- indexes and partial indexes
- `jsonb`
- transactions
- optimistic concurrency
- Row-Level Security
- migrations and seed data

Provider-specific features are used only where they provide a concrete benefit.

---

## Reference Data (fachliches Datenkonzept)

Reference Data is treated separately from normal Aggregate persistence.

The architecture distinguishes between:

```text
Domain Aggregate
        │
        └── normal lifecycle

Reference / Lookup Data
        │
        └── controlled lifecycle

Code-defined Enum Entity
        │
        └── stable identity + seed data
```

Example:

```text
OrderStatus
┌────┬───────────┬─────────┐
│ Id │ Name      │ IsFinal │
├────┼───────────┼─────────┤
│ 1  │ Pending   │ false   │
│ 2  │ Confirmed │ false   │
│ 3  │ Shipped   │ true    │
│ 4  │ Cancelled │ true    │
└────┴───────────┴─────────┘
```

Code-defined values are synchronized with PostgreSQL through controlled seed/migration mechanisms.

A persisted enum value must keep its stable identity once released.

---

## Caching

Reference Data is a natural candidate for caching because it is typically:

- read frequently
- relatively small
- stable
- changed in a controlled manner

The preferred conceptual model is cache-aside:

```text
Application
     │
     ▼
   Cache
   ┌─┴─┐
  Hit Miss
   │    │
   │    ▼
   │ Repository
   │    │
   │    ▼
   │ PostgreSQL
   │    │
   └────┘
```

Concurrent cache misses are coordinated so that:

```text
5 concurrent requests
        │
        ▼
1 database query
        │
        ▼
5 results
```

For multiple application instances, Redis can provide a shared distributed cache.

---

## Multi-Tenancy

Multi-Tenancy extends the base architecture:

```text
Single-Tenant

Domain
  │
  ▼
Aggregate
  │
  ▼
Repository
  │
  ▼
PostgreSQL
```

versus:

```text
Multi-Tenant

Request
  │
  ▼
Tenant Resolution
  │
  ▼
Tenant Context
  │
  ▼
Application
  │
  ▼
Repository / EF Core
  │
  ▼
Tenant Isolation
  │
  ▼
PostgreSQL
```

The architecture separates:

```text
Tenant Resolution
Tenant Context
Tenant Filtering
Tenant Authorization
Tenant Isolation
```

Tenant isolation is treated as a **security requirement**.

Defense-in-depth can combine:

```text
Application Authorization
        ↓
EF Core Filtering
        ↓
Database Constraints
        ↓
PostgreSQL Row-Level Security
```

The final architecture determines which layers are appropriate for the deployment scenario.

---

## CQRS & Queries

CQRS is applied pragmatically.

Commands typically operate through aggregates:

```text
Command
   │
   ▼
Aggregate Repository
   │
   ▼
Aggregate
   │
   ▼
EF Core
```

Queries may use dedicated read models:

```text
Query
   │
   ▼
Query Handler
   │
   ├── EF Core
   └── Dapper
   │
   ▼
DTO / Projection
```

Not every read needs to pass through an Aggregate Repository.

Complex projections, reporting and read-heavy scenarios may justify dedicated query infrastructure.

---

## Repository Structure

The repository intentionally avoids a universal hierarchy such as:

```text
IRepository
 ├── ICacheRepository
 ├── ITenantRepository
 ├── IAuditRepository
 ├── IEnumRepository
 └── IChildEntityRepository
```

Instead, responsibilities remain separate:

```text
Aggregate Persistence
        │
        ▼
Aggregate Repository

Reference Data
        │
        ▼
Reference Data Access

Tenant Resolution
        │
        ▼
Tenant Context

Auditing
        │
        ▼
EF Core Interceptor

Caching
        │
        ▼
Cache Infrastructure
```

This distinction is important:

> **A cache is not automatically a repository, and a context or interceptor is not a repository.**

---

## Repository Structure

```text
.
├── LICENSE
├── README.md
│
└── prompts
    └── postgres
        └── 05-architecture-design-postgres.md
```

The repository is intentionally structured so that the **architecture prompt** can evolve independently from the eventual reference implementation.

---

## Architecture Prompt

The detailed architecture specification is maintained in:

**[`prompts/postgres/05-architecture-design-postgres.md`](prompts/postgres/05-architecture-design-postgres.md)**

The prompt covers:

- architecture and layer boundaries
- Entity and Aggregate design
- Strongly Typed IDs
- Entity Equality
- Child Entities
- optional Multi-Tenancy
- tenant isolation and PostgreSQL RLS
- auditing
- Reference Data and Enum Entities
- repositories
- caching and cache stampede protection
- Redis and multi-instance deployments
- EF Core configuration
- PostgreSQL / Npgsql
- concurrency
- CQRS
- Specification Pattern
- Unit of Work
- dependency injection
- testing
- security
- PlantUML architecture diagrams
- final decision matrices

The README intentionally remains concise; the prompt contains the detailed architectural analysis and implementation requirements.

---

## Status

🚧 **Architecture / Reference Project**

The repository is evolving from the architecture specification toward a concrete reference implementation.

Architectural decisions should be:

1. explicit,
2. justified,
3. documented,
4. validated through tests,
5. changed deliberately.

The project does not aim to provide a universal architecture for every .NET application.

---

## License

This project is licensed under the terms defined in [`LICENSE`](LICENSE).

---

## Guiding Principle

> **Keep the core architecture simple. Add complexity only when the business or technical requirements justify it.**

In particular:

```text
                    BASE ARCHITECTURE
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
        Audit        Multi-Tenancy       Caching
       optional         optional         optional
```

**Single-Tenancy remains the simple default. Multi-Tenancy, Auditing and Caching are capabilities — not prerequisites for the domain model.**