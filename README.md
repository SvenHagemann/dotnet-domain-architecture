# .NET Domain Architecture

A reusable architecture blueprint and AI-assisted design prompt for building clean, generic, and extensible .NET applications based on Domain-Driven Design (DDD), Entity Framework Core, and a layered architecture.

The repository provides a structured approach for designing:

- Generic domain entities
- Strongly typed IDs
- Aggregate Roots and Child Entities
- Repository abstractions
- Reference Data and persistable enum-based entities
- Auditing
- Optional multi-tenancy
- Reference-data caching
- EF Core configurations
- Unit of Work
- CQRS and query handling
- Dependency Injection
- PlantUML architecture diagrams

The architecture is designed with **Single-Tenant applications as the simple baseline**, while **Multi-Tenancy is treated as an optional capability** that can be added when required.

---

## Goals

The goal of this project is to provide a consistent foundation for designing .NET applications where domain concepts, persistence, caching, and infrastructure concerns remain clearly separated.

The architecture focuses on the following principles:

- **Domain-first design**
- **Explicit Aggregate Boundaries**
- **Repositories at Aggregate boundaries**
- **Child Entities without independent repositories**
- **Strongly Typed IDs**
- **Composition over unnecessary inheritance**
- **Optional cross-cutting capabilities**
- **Clear separation of Domain, Application, and Infrastructure**
- **EF Core as a persistence implementation detail**
- **Caching separated from persistence**
- **Reference Data optimized for read-heavy access**
- **Multi-Tenancy without forcing tenant awareness onto every entity**

---

## Architecture Overview

The conceptual architecture looks like this:

```text
                         ┌─────────────────────┐
                         │       Domain        │
                         │                     │
                         │ Entities            │
                         │ Aggregates          │
                         │ Value Objects       │
                         │ Domain Rules        │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │     Application     │
                         │                     │
                         │ Commands            │
                         │ Queries             │
                         │ Use Cases           │
                         │ Repository Contracts│
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Infrastructure    │
                         │                     │
                         │ EF Core             │
                         │ Repositories        │
                         │ Database            │
                         │ Cache               │
                         │ Redis               │
                         │ Interceptors        │
                         └─────────────────────┘
```

Cross-cutting capabilities such as auditing, caching, and multi-tenancy are introduced where they are actually required instead of becoming mandatory dependencies of every entity.

---

## Single-Tenant First

A central design principle is that the architecture must work naturally without Multi-Tenancy.

A basic application should be able to use:

```text
Entity
   │
   ▼
Aggregate Root
   │
   ▼
Repository
   │
   ▼
EF Core
   │
   ▼
Database
```

without requiring:

```text
TenantId
ITenantContext
ITenantEntity
TenantRepository
Tenant Query Filters
Tenant-aware Cache Keys
```

Multi-Tenancy can then be introduced as an additional capability:

```text
                         Entity
                            │
                    ┌───────┴───────┐
                    │               │
                 Aggregate        optional
                    │             capabilities
                    │          ┌────┼────┐
                    │          │    │    │
                    ▼        Audit Tenant Cache
                Repository
                    │
                    ▼
                  EF Core
                    │
                    ▼
                 Database
```

This keeps the core domain model simple while allowing the same architecture to support multi-tenant applications.

---

## Aggregate Boundaries

Aggregates define consistency and persistence boundaries.

For example:

```text
Order
 ├── OrderLine
 ├── OrderLine
 └── ShippingAddress
```

`Order` is the Aggregate Root.

`OrderLine` and `ShippingAddress` belong to the aggregate and therefore do not require their own repositories.

The intended access pattern is:

```text
IOrderRepository
       │
       ▼
     Order
       │
       ├── OrderLine
       ├── OrderLine
       └── ShippingAddress
```

rather than:

```text
IOrderRepository
IOrderLineRepository
IShippingAddressRepository
```

This makes the Aggregate Root the primary entry point for modifying its internal state and enforcing domain invariants.

---

## Strongly Typed IDs

The architecture favors strongly typed identifiers over primitive IDs.

For example:

```csharp
public readonly record struct OrderId(Guid Value);

public readonly record struct OrderLineId(Guid Value);

public readonly record struct TenantId(Guid Value);

public readonly record struct UserId(Guid Value);
```

This prevents accidental mixing of identifiers:

```csharp
void ProcessOrder(OrderId orderId)
{
}
```

A `CustomerId` cannot accidentally be passed where an `OrderId` is expected.

Strongly typed IDs are integrated with the generic entity model and EF Core persistence.

---

## Reference Data and Enum Entities

This architecture distinguishes between ordinary Lookup Entities and **persistable enum-based reference data**.

A persistable enum entity derives its identity from a C# enum:

```csharp
public enum OrderStatus
{
    Pending = 1,
    Confirmed = 2,
    Shipped = 3,
    Cancelled = 4
}
```

The numeric value becomes the database identity:

```text
OrderStatus
┌────┬───────────┬─────────┬────────────────────┐
│ Id │ Name      │ IsFinal │ Additional Metadata│
├────┼───────────┼─────────┼────────────────────┤
│ 1  │ Pending   │ false   │ ...                │
│ 2  │ Confirmed │ false   │ ...                │
│ 3  │ Shipped   │ true    │ ...                │
│ 4  │ Cancelled │ true    │ ...                │
└────┴───────────┴─────────┴────────────────────┘
```

This allows additional domain-specific metadata to be persisted alongside the enum value.

Examples include:

- `IsFinal`
- `AllowsModification`
- `SortOrder`
- `RequiresApproval`
- `IsActive`

These entities are treated as **reference data**, not as normal CRUD entities.

---

## Reference Data Caching

Reference data is generally read much more frequently than it changes.

Therefore, the architecture uses a cache-aside approach:

```text
First request

Application
    │
    ▼
  Cache
    │
   Miss
    │
    ▼
Database
    │
    ▼
  Cache
    │
    ▼
Response
```

Subsequent requests:

```text
Application
    │
    ▼
  Cache
    │
   Hit
    │
    ▼
Response
```

The implementation should also prevent multiple concurrent requests from causing the same database query during a cache miss.

For example:

```text
5 concurrent requests
        │
        ▼
   Cache Miss
        │
        ▼
  1 Database Query
        │
        ▼
      Cache
        │
   ┌────┼────┬────┬────┐
   ▼    ▼    ▼    ▼    ▼
   A    B    C    D    E
```

The cache itself remains an infrastructure concern and is deliberately separated from the repository/persistence abstraction.

---

## Repository Architecture

Repositories operate primarily at Aggregate boundaries.

A conceptual structure is:

```text
IRepository<TAggregate, TKey>
            │
            ├── IOrderRepository
            │
            └── OrderRepository
```

Reference data follows a different access pattern:

```text
IReferenceDataRepository<T, TKey>
            │
            ▼
       EF Core
```

with caching placed in front of it:

```text
Application
     │
     ▼
ReferenceDataService
     │
     ▼
ReferenceDataCache
     │
   Hit/Miss
     │
     ▼
ReferenceDataRepository
     │
     ▼
   EF Core
     │
     ▼
  Database
```

The project deliberately evaluates whether concepts such as `CacheRepository` or `TenantRepository` are actually repositories or whether they are better modeled as services, contexts, decorators, or infrastructure components.

---

## Auditing

Auditing is treated as an optional capability.

Typical audit information includes:

```text
CreatedAt
CreatedBy
ModifiedAt
ModifiedBy
```

Rather than creating a large inheritance hierarchy such as:

```text
AuditableEntity
TenantEntity
AuditableTenantEntity
AuditableAggregateRoot
TenantAggregateRoot
...
```

the architecture favors interfaces and infrastructure mechanisms where appropriate.

Audit values can be populated automatically during persistence, for example through an EF Core `SaveChangesInterceptor`.

---

## Multi-Tenancy

Multi-Tenancy is an optional capability.

A tenant-aware entity may implement:

```csharp
public interface ITenantEntity<TTenantKey>
    where TTenantKey : struct
{
    TTenantKey TenantId { get; }
}
```

while a normal single-tenant entity does not need to know anything about tenants.

For multi-tenant applications, the architecture considers:

- Tenant resolution
- `ITenantContext`
- Tenant-aware repositories
- EF Core global query filters
- Tenant isolation
- Database constraints
- Composite keys
- Tenant-aware cache keys
- Optional database Row-Level Security

The important distinction is:

```text
Tenant Resolution
       ≠
Tenant Filtering
       ≠
Tenant Authorization
       ≠
Tenant Data Isolation
```

These concerns should not automatically be collapsed into a single abstraction.

---

## EF Core

EF Core is treated as part of the Infrastructure/Persistence layer.

Entity configuration follows a generic configuration hierarchy where appropriate:

```text
EntityConfiguration<T, TKey>
        │
        ├── Aggregate Configuration
        ├── Child Entity Configuration
        ├── Enum Entity Configuration
        ├── optional Tenant Configuration
        └── optional Audit Configuration
```

The goal is to reuse common configuration logic without creating a combinatorial explosion of specialized configuration classes.

For example:

```text
Entity
Aggregate Root
Aggregate Root + Audit
Aggregate Root + Tenant
Aggregate Root + Tenant + Audit
```

should not necessarily result in four separate configuration inheritance branches.

---

## Example Domain

The example domain used throughout the architecture is an Order aggregate:

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

with reference data such as:

```text
OrderStatus
PaymentType
Country
```

This domain demonstrates:

- Aggregate boundaries
- Child entities
- Strongly typed IDs
- Reference data
- Persistable enum entities
- Repository access
- Caching
- Auditing
- Optional Multi-Tenancy
- EF Core configuration

---

## Repository Structure

The repository is organized as follows:

```text
dotnet-domain-architecture/
│
├── README.md
│
├── prompts/
│   └── architecture-design.md
│
├── diagrams/
│   ├── entity-model.puml
│   ├── aggregate.puml
│   ├── repositories.puml
│   ├── caching.puml
│   └── tenancy.puml
│
├── examples/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│
└── docs/
    ├── architecture.md
    ├── aggregates.md
    ├── repositories.md
    ├── reference-data.md
    ├── caching.md
    └── multi-tenancy.md
```

The exact structure may evolve as the architecture develops.  

The architectural decisions resulting from the prompt:
```text
prompts/architecture-design.md
```
are located in `docs/`.

---

## AI Architecture Prompt

The primary purpose of the `prompts/` directory is to provide a reusable prompt for AI-assisted architecture design.

The prompt asks an AI model to:

1. Analyze the domain model.
2. Design generic entities and interfaces.
3. Define Aggregate boundaries.
4. Design repository abstractions.
5. Design optional Audit and Multi-Tenancy capabilities.
6. Design persistable enum/reference data.
7. Design reference-data caching.
8. Design EF Core configurations.
9. Produce production-oriented C# code.
10. Produce PlantUML diagrams.
11. Explain important architectural decisions.
12. Compare alternatives and explicitly justify the final recommendation.

The prompt is intentionally detailed so that architectural decisions are not made in isolation.

---

## Design Principles

The architecture follows these general principles:

### 1. Aggregates define repository boundaries

Repositories should normally operate on Aggregate Roots rather than individual child entities.

### 2. Domain should not depend on infrastructure

The domain model should not depend directly on:

- EF Core
- Redis
- SQL
- HTTP
- ASP.NET Core
- infrastructure-specific caching implementations

### 3. Capabilities should remain optional

Audit, Multi-Tenancy, and caching should only introduce dependencies where they are actually required.

### 4. Persistence should not dictate the domain model

EF Core configuration should adapt the persistence model to the domain rather than forcing infrastructure concerns into domain abstractions.

### 5. Reference data is different from normal CRUD data

Reference data often has different lifecycle, caching, seeding, and modification requirements.

### 6. Caching is not persistence

A cache is an optimization and must not become the authoritative source of domain state.

### 7. Prefer explicit boundaries over generic abstractions everywhere

Generic abstractions should provide real value. A generic repository, cache repository, or tenant repository should not exist merely because it can be made generic.

---

## Status

This repository is intended as an evolving architecture blueprint rather than a framework or NuGet package.

The examples and recommendations may change as different architectural alternatives are evaluated.

The primary goals are:

- clarity
- consistency
- type safety
- maintainability
- testability
- scalability
- explicit architectural boundaries

---

## Use of the prompt



### Use a two-stage approach


```text
┌─────────────────────────────┐
│ 1. PLAN                     │
│                             │
│ Analyze requirements        │
│ Design the architecture     │
│ Evaluate alternatives       │
│ Define architectural        │
│ decisions                   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ 2. AGENT                    │
│                             │
│ Implement C#                │
│ Implement EF Core           │
│ Implement repositories      │
│ Implement caching           │
│ Create PlantUML diagrams    │
│ Create tests                │
│ Create documentation        │
└─────────────────────────────┘
```

**Plan** is therefore an excellent choice for the first iteration, while you are still discussing and refining the architecture.

Once you say:

> "The architecture looks good as it is. Implement it."

→ **Agent**.

I would primarily use **Ask** for subsequent, more detailed questions, for example:

> "Why is `IReferenceDataCache<T,TKey>` better than `ICacheRepository<T,TKey>`?"

or:

> "Should `TenantId` be part of the primary key?"

**Plan → Review → Agent** is, in my opinion, the best workflow.