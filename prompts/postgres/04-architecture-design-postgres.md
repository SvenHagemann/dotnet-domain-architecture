
## ASP.NET Core / EF Core / PostgreSQL / DDD / optionale Multi-Tenancy

---

# 0. Rolle und Ziel der Aufgabe

Du bist ein erfahrener Softwarearchitekt mit Schwerpunkt auf:

- C#
- .NET / ASP.NET Core
- Entity Framework Core
- PostgreSQL / Npgsql
- Domain-Driven Design
- Clean Architecture
- Modularisierung
- Persistence Architecture
- Multi-Tenancy
- Caching
- CQRS
- Testbarkeit
- langfristig wartbaren Enterprise-Anwendungen

Entwirf eine **konsistente, produktionsgeeignete Gesamtarchitektur** für eine verteilte E-Commerce-Anwendung auf Basis von:

- C#
- ASP.NET Core
- Entity Framework Core
- PostgreSQL
- Npgsql

Die Architektur soll langfristig wartbar, testbar, typsicher und praxisnah sein.

Behandle PostgreSQL als **konkreten Persistence Provider**. Provider-spezifische Entscheidungen sollen daher nicht abstrakt bleiben, sondern anhand von EF Core + Npgsql + PostgreSQL bewertet werden.

Die E-Commerce-Anwendung besteht unter anderem aus folgenden fachlichen Services bzw. Modulen:

```text
Basket
Catalog
Discount
Ordering
Payment
Auth
```

Für die konkrete Demonstration soll primär die **Ordering-/Order-Domain** verwendet werden.

---

# 1. Zentrales Architekturprinzip

Die wichtigste Anforderung lautet:

> **Single-Tenancy ist der einfache und vollständige Basisfall. Multi-Tenancy ist eine optionale Capability, die additiv aktiviert werden kann.**

Die Basisarchitektur darf daher nicht von Multi-Tenancy abhängen.

Konzeptionell:

```text
                    ┌──────────────────────────┐
                    │     Basisarchitektur     │
                    │                          │
                    │ Entity                   │
                    │ Aggregate Root           │
                    │ Child Entity             │
                    │ Repository               │
                    │ EF Core                  │
                    │ PostgreSQL               │
                    └────────────┬─────────────┘
                                 │
                       optionale Capabilities
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
            Audit           Multi-Tenancy        Caching
```

Die Architektur muss vollständig ohne Multi-Tenancy funktionieren.

Eine Single-Tenant-Anwendung soll insbesondere **nicht zwingend benötigen**:

```text
ITenantContext
ITenantEntity<TTenantKey>
TenantId
TenantRepository
Tenant Query Filters
Tenant Resolution
Tenant-aware Cache Keys
Tenant-specific Infrastructure
```

wenn diese fachlich nicht erforderlich sind.

Eine Multi-Tenant-Anwendung soll dagegen dieselbe Basisarchitektur verwenden können und nur die tatsächlich erforderlichen zusätzlichen Komponenten aktivieren.

---

# 2. Gewünschtes Ergebnis

Entwickle keine isolierte Sammlung von Klassen.

Entwickle stattdessen eine **zusammenhängende Architektur** und beantworte dabei:

1. Welche Abstraktionen existieren?
2. In welcher Schicht liegen sie?
3. Welche Abstraktionen sind Basiskomponenten?
4. Welche sind optionale Capabilities?
5. Welche Abstraktionen sind tatsächlich Repositorys?
6. Welche sollten besser Services, Contexts, Decorators oder Infrastructure Components sein?
7. Wie arbeiten Domain, Application und Infrastructure zusammen?
8. Wie funktioniert die Architektur ohne Multi-Tenancy?
9. Wie wird Multi-Tenancy additiv aktiviert?
10. Wie wird PostgreSQL konkret eingebunden?
11. Wie werden Strongly Typed IDs persistiert?
12. Wie werden Aggregate und Child Entities persistiert?
13. Wie werden Reference Data und persistierbare Enum Entities behandelt?
14. Wie wird Reference Data gecacht?
15. Wie werden Cache Stampedes verhindert?
16. Wie wird Audit umgesetzt?
17. Wann ist CQRS sinnvoll?
18. Wann ist ein Specification Pattern sinnvoll?
19. Ist eine zusätzliche Unit of Work erforderlich?
20. Wie wird die Architektur getestet?

Treffe am Ende eine **eindeutige Empfehlung für eine produktionsgeeignete Lösung**.

Alternativen sollen dargestellt werden, aber die finale Architektur darf nicht in einem "kommt darauf an" enden.

---

# 3. Architekturprinzipien

Berücksichtige insbesondere:

- Domain-Driven Design
- Aggregate Boundaries
- Clean Architecture
- Separation of Concerns
- Dependency Inversion
- Strongly Typed IDs
- Encapsulation
- Composition over Inheritance
- optionale Cross-Cutting Capabilities
- testbare Persistence
- EF-Core-Praxis
- PostgreSQL-spezifische Möglichkeiten
- horizontale Skalierung
- Multi-Instance Deployment
- Security by Design

Vermeide unnötige Abstraktionen.

Insbesondere gilt:

> Nicht jede technische Komponente benötigt ein Interface, und nicht jede Abstraktion ist automatisch ein Repository.

Bewerte jede wesentliche Abstraktion kritisch.

---

# 4. Ziel-Schichtenmodell

Untersuche und empfehle eine klare Schichtung, beispielsweise:

```text
┌───────────────────────────────────────────────┐
│                    Domain                     │
│                                               │
│ Entities                                      │
│ Aggregate Roots                               │
│ Child Entities                                │
│ Value Objects                                 │
│ Domain Rules                                  │
│ Domain Interfaces                             │
└────────────────────────┬──────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────┐
│                 Application                   │
│                                               │
│ Use Cases                                     │
│ Commands                                      │
│ Queries                                       │
│ Application Services                          │
│ Repository Interfaces                         │
│ Reference Data Interfaces                     │
│ optional Tenant Abstractions                  │
└────────────────────────┬──────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────┐
│                Infrastructure                 │
│                                               │
│ EF Core                                       │
│ Npgsql                                        │
│ PostgreSQL                                    │
│ Repository Implementations                    │
│ Cache                                         │
│ Redis / Distributed Cache                     │
│ EF Interceptors                               │
│ Tenant Resolution                             │
│ Database-specific Infrastructure              │
└───────────────────────────────────────────────┘
```

Begründe genau, welche Abstraktion in welche Schicht gehört.

Besonders wichtig:

Die Domain darf nicht von folgenden technischen Komponenten abhängen:

```text
EF Core
Npgsql
PostgreSQL
Redis
IMemoryCache
IDistributedCache
ASP.NET Core
HttpContext
Tenant Resolution Infrastructure
```

Falls eine Abstraktion in der Domain oder Application benötigt wird, soll sie über eine geeignete Abstraktion erfolgen.

---

# 5. Entity-Modell

Bewerte zunächst folgende Ausgangsidee:

```csharp
public interface IEntity<TKey> : IDomainObject
    where TKey : struct
{
    TKey Id { get; }
}

public abstract class Entity<TKey> :
    IEntity<TKey>,
    IEquatable<Entity<TKey>>
    where TKey : struct
{
    public virtual TKey Id { get; init; }

    public virtual bool IsNew =>
        EqualityComparer<TKey>.Default.Equals(Id, default);

    public override int GetHashCode() =>
        Id.GetHashCode();

    public static bool operator ==(
        Entity<TKey> left,
        Entity<TKey> right) =>
        Equals(left, right);

    public static bool operator !=(
        Entity<TKey> left,
        Entity<TKey> right) =>
        !(left == right);

    public override string ToString() =>
        $"[{GetType().Name} {Id}]";

    public override bool Equals(object? obj)
    {
        if (obj == null || obj is not Entity<TKey>)
            return false;

        return Equals((Entity<TKey>)obj);
    }

    public bool Equals(Entity<TKey>? other)
    {
        if (other is null)
            return false;

        if (ReferenceEquals(this, other))
            return true;

        if (IsNew && other.IsNew)
            return false;

        var thisType = GetType();
        var otherType = other.GetType();

        if (!thisType.IsAssignableFrom(otherType) &&
            !otherType.IsAssignableFrom(thisType))
            return false;

        return Id.Equals(other.Id);
    }
}
```

Bewerte diese Implementierung kritisch.

Untersuche insbesondere:

- Entity Equality
- Identity
- transient Entities
- HashCode
- polymorphe Entities
- Vererbung
- EF-Core-Proxies
- Proxy-Typen
- Strongly Typed IDs
- `init`
- `private set`
- Persistence Concerns
- Domain Concerns
- Collections
- Dictionary Keys
- Änderung der ID nach Einfügen in eine Collection
- Equality zwischen Proxy und konkreter Entity
- Equality über Basisklassen
- Verhalten bei nicht gesetzten IDs

Entscheide, ob eine Entity-Basisklasse überhaupt Equality implementieren sollte.

Falls ja, liefere eine robuste Implementierung.

Falls nein, erkläre warum.

Die finale Lösung muss klar zwischen:

```text
Domain Identity
```

und:

```text
Persistence Mechanics
```

unterscheiden.

---

# 6. Strongly Typed IDs

Verwende Strongly Typed IDs.

Beispielsweise:

```csharp
public readonly record struct OrderId(Guid Value);

public readonly record struct OrderLineId(Guid Value);

public readonly record struct TenantId(Guid Value);

public readonly record struct UserId(Guid Value);

public readonly record struct OrderStatusId(int Value);
```

Untersuche:

- `record struct`
- `readonly record struct`
- `struct`
- Value Object
- Generic Constraints
- EF Core Value Converters
- Npgsql
- PostgreSQL `uuid`
- Integer Keys
- Foreign Keys
- Composite Keys
- Indexes
- Performance
- Typsicherheit

Zeige insbesondere, wie verhindert wird, dass:

```csharp
CustomerId
```

versehentlich an eine Methode übergeben wird, die:

```csharp
OrderId
```

erwartet.

Beurteile außerdem:

```text
Guid
UUID
int
bigint
```

als zugrunde liegende PostgreSQL-Typen.

Treffe eine konkrete Empfehlung.

---

# 7. Aggregate Roots

Entwickle eine generische Struktur für Aggregate Roots.

Ausgangspunkt:

```csharp
public interface IAggregateRoot<TKey> : IEntity<TKey>
    where TKey : struct
{
}

public abstract class AggregateRoot<TKey> :
    Entity<TKey>,
    IAggregateRoot<TKey>
    where TKey : struct
{
}
```

Bewerte diese Struktur.

Erkläre:

- Was ist ein Aggregate Root?
- Was unterscheidet es von einer Entity?
- Warum ist das Aggregate die Repository-Grenze?
- Warum benötigt nicht jede Entity ein Repository?
- Wie kann das Type-System Aggregate Roots kennzeichnen?
- Welche Fehler können Generic Constraints verhindern?
- Welche Fehler kann C# nicht verhindern?

Die finale Repository-Architektur darf Child Entities nicht als eigenständige Aggregate behandeln.

---

# 8. Child Entities

Verwende als Beispiel:

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

`Order` ist Aggregate Root.

`OrderLine` gehört ausschließlich zum Aggregate.

`ShippingAddress` soll kritisch als:

```text
Child Entity
```

oder:

```text
Value Object / Owned Entity
```

bewertet werden.

Begründe die Entscheidung.

Falls ein Child-Entity-Typ sinnvoll ist, untersuche:

```csharp
public interface IChildEntity<TKey> : IEntity<TKey>
    where TKey : struct
{
}

public abstract class ChildEntity<TKey> :
    Entity<TKey>,
    IChildEntity<TKey>
    where TKey : struct
{
}
```

Bewerte, ob hierfür eine Basisklasse erforderlich ist oder ein Interface ausreicht.

Es darf kein:

```csharp
IOrderLineRepository
```

geben.

---

# 9. Aggregate Encapsulation

Aggregate sollen ihre Children selbst verwalten.

Bevorzugt:

```csharp
order.AddLine(...);

order.RemoveLine(...);

order.ChangeShippingAddress(...);
```

nicht:

```csharp
order.OrderLines.Add(...);

order.OrderLines.Remove(...);
```

Zeige eine geeignete Kapselung:

```csharp
private readonly List<OrderLine> _lines = [];

public IReadOnlyCollection<OrderLine> Lines =>
    _lines;
```

Bewerte:

- Invarianten
- Domain Methods
- Child Lifecycle
- Navigation Properties
- EF Core Compatibility
- Backing Fields
- Collection Mapping
- Cascade Delete
- Aggregate Boundary

---

# 10. Optionale Capabilities

Behandle folgende Themen als voneinander unabhängige optionale Capabilities:

```text
Audit
Multi-Tenancy
Caching
```

Vermeide eine Vererbungshierarchie wie:

```text
Entity
AuditableEntity
TenantEntity
AuditableTenantEntity
AggregateRoot
AuditableAggregateRoot
TenantAggregateRoot
AuditableTenantAggregateRoot
...
```

Bevorzuge, sofern sinnvoll:

```text
Interfaces
Composition
Infrastructure Services
Configuration Components
Decorators
```

Begründe die jeweilige Entscheidung.

---

# 11. Multi-Tenancy

Multi-Tenancy muss optional sein.

Eine mögliche Abstraktion:

```csharp
public interface ITenantEntity<TTenantKey>
    where TTenantKey : struct
{
    TTenantKey TenantId { get; }
}
```

Bewerte diese Lösung kritisch.

Untersuche:

- Interface
- Basisklasse
- Composition
- Generic Constraint
- Tenant Context
- Application Service
- Repository Decorator
- EF Core Global Query Filter
- EF Core Interceptor
- PostgreSQL Row-Level Security
- Database Constraints

Eine Entity soll weiterhin problemlos so aussehen können:

```csharp
public sealed class Product
    : AggregateRoot<ProductId>
{
}
```

ohne Tenant-Abhängigkeit.

Eine tenant-aware Entity könnte beispielsweise sein:

```csharp
public sealed class Order :
    AggregateRoot<OrderId>,
    ITenantEntity<TenantId>
{
}
```

Entscheide selbst, ob diese Lösung optimal ist.

---

# 12. Tenant-Verantwortlichkeiten

Unterscheide ausdrücklich:

```text
Tenant Resolution
Tenant Context
Tenant Filtering
Tenant Authorization
Tenant Isolation
```

Definiere für jede Verantwortung die zuständige Schicht.

Beispielsweise:

```text
HTTP Request
    │
    ▼
Tenant Resolution
    │
    ▼
ITenantContext
    │
    ▼
Application
    │
    ▼
EF Core / Repository
    │
    ▼
Tenant Filtering
    │
    ▼
PostgreSQL
```

Erkläre, warum Tenant Resolution nicht dasselbe ist wie Tenant Authorization oder Tenant Isolation.

---

# 13. Tenant Isolation

Behandle Tenant Isolation als **Security Requirement**.

Ein Request aus Tenant A darf niemals Daten von Tenant B lesen oder verändern.

Beispiel:

```text
Tenant A
 └── Order A

Tenant B
 └── Order B
```

Tenant A darf niemals:

```text
Order B
```

erhalten.

Bewerte mehrere Schutzschichten:

1. Application Layer
2. Repository
3. EF Core Global Query Filters
4. EF Core Interceptors
5. Database Constraints
6. PostgreSQL Row-Level Security
7. Authorization

Erkläre, welche Schichten Defense-in-Depth liefern und welche die eigentliche Sicherheitsgrenze bilden.

Bewerte insbesondere PostgreSQL Row-Level Security als zusätzliche Sicherheitsmaßnahme.

Treffe eine konkrete Empfehlung.

---

# 14. TenantId und Datenbank-Schlüssel

Vergleiche:

### Variante A

```text
PK = EntityId
TenantId = separate column
```

### Variante B

```text
PK = TenantId + EntityId
```

Bewerte:

- Primary Keys
- Foreign Keys
- Composite Keys
- Unique Constraints
- Indexes
- Query Performance
- EF Core Mapping
- Aggregate References
- Cross-Tenant References
- PostgreSQL
- Sicherheit
- Migrationen

Entscheide dich für eine Standardstrategie.

Erkläre auch, wann davon abgewichen werden sollte.

---

# 15. Audit

Audit ist ebenfalls optional.

Untersuche:

```csharp
public interface IAuditableEntity<TUserKey>
    where TUserKey : struct
{
    DateTime CreatedAt { get; }
    TUserKey CreatedBy { get; }

    DateTime? ModifiedAt { get; }
    TUserKey? ModifiedBy { get; }
}
```

Bewerte:

- Interface
- Basisklasse
- Composition
- EF Core Interceptor
- `SaveChanges`
- `SaveChangesAsync`
- Current User Context
- Time Provider
- UTC
- Domain vs. Infrastructure

Bevorzuge eine Lösung, bei der Audit nicht die Entity-Hierarchie vervielfacht.

Zeige konkrete Implementierung.

---

# 16. Persistierbare Enum Entities

Definiere präzise, was mit einer "persistierbaren Enum Entity" gemeint ist.

Es handelt sich ausdrücklich **nicht um eine gewöhnliche Lookup Entity**.

Beispiel:

```csharp
public enum OrderStatus
{
    Pending = 1,
    Confirmed = 2,
    Shipped = 3,
    Cancelled = 4
}
```

Die persistierte Struktur:

```text
OrderStatus
┌────┬───────────┬─────────┬────────────────────┐
│ Id │ Name      │ IsFinal │ weitere Metadaten  │
├────┼───────────┼─────────┼────────────────────┤
│ 1  │ Pending   │ false   │ ...                │
│ 2  │ Confirmed │ false   │ ...                │
│ 3  │ Shipped   │ true    │ ...                │
│ 4  │ Cancelled │ true    │ ...                │
└────┴───────────┴─────────┴────────────────────┘
```

Dabei gilt:

- `Id` entspricht dem numerischen Enum-Wert.
- `Id` ist Primary Key.
- `Id` ist stabil.
- EF Core darf `Id` nicht generieren.
- `ValueGeneratedNever()` soll verwendet werden.
- `Name` wird explizit persistiert.
- zusätzliche fachliche Metadaten sind möglich.
- Benutzer dürfen keine Werte über normales CRUD anlegen.
- Werte werden durch Code definiert.
- Werte werden über Seed Data persistiert.
- andere Entities können über den stabilen Schlüssel referenzieren.
- es gibt kein normales CRUD-Repository.

---

# 17. Enum Entity vs. Lookup Entity

Erkläre den Unterschied zwischen:

```text
Persistierbare Enum Entity
```

und:

```text
Lookup / Reference Data Entity
```

Beispiele:

```text
Enum Entity:
OrderStatus
PaymentType
```

gegebenenfalls:

```text
Lookup:
Country
Currency
CustomerCategory
```

Eine Lookup Entity kann fachlich gegebenenfalls administrativ gepflegt werden.

Eine Enum Entity wird dagegen durch Code definiert.

Bewerte, ob `PaymentType` tatsächlich eine Enum Entity sein sollte oder eher Reference Data.

Treffe diese Entscheidung fachlich begründet.

---

# 18. Ansätze für Enum Entities

Vergleiche mindestens:

### Ansatz A

```text
C# Enum + separate EF Entity
```

### Ansatz B

```text
Smart Enum / Enumeration Pattern
```

### Ansatz C

```text
Generic EnumEntity<TKey,TEnum>
```

Bewerte:

- Typsicherheit
- numerische Identität
- String-Repräsentation
- Metadaten
- EF Core
- PostgreSQL
- Seed Data
- Migrationen
- Testbarkeit
- Cache
- Performance
- Synchronisierung

Treffe eine konkrete Empfehlung.

---

# 19. Enum-Synchronisierung

Definiere eine robuste Strategie für:

```text
C# Enum
      ↕
Seed Data
      ↕
PostgreSQL
```

Bewerte:

- EF Core `HasData`
- Migration Seed
- Custom Seed Extensions
- Startup Validation
- Integration Tests
- Build-Time Validation
- Runtime Validation

Behandle insbesondere:

```text
C#:
Shipped = 3

Database:
Shipped = 4
```

und:

```text
C#:
Cancelled = 4

Database:
Cancelled fehlt
```

Definiere:

- welche Fälle erlaubt sind
- welche Fehler erkannt werden
- wann Fehler erkannt werden
- ob die Anwendung starten darf
- ob Migrationen notwendig sind
- wie Änderungen an Enum-Werten behandelt werden

Ein einmal produktiv verwendeter Enum-Wert darf nicht einfach umnummeriert werden.

---

# 20. Repository-Architektur

Bewerte kritisch:

```text
IRepository
    ├── ICacheRepository
    ├── ITenantRepository
    ├── IAuditRepository
    ├── IEnumRepository
    ├── IAggregateRepository
    └── IChildEntityRepository
```

Diese Struktur soll **nicht automatisch übernommen werden**.

Beantworte ausdrücklich:

> Welche Abstraktionen sind tatsächlich Repositorys und welche sollten besser Services, Contexts, Decorators oder Infrastructure Components sein?

Eine mögliche Zielstruktur:

```text
Aggregate Persistence
        │
        ▼
IRepository<TAggregate,TKey>

Reference Data
        │
        ▼
IReferenceDataRepository<T,TKey>

Caching
        │
        ▼
IReferenceDataCache<T,TKey>

Tenant Resolution
        │
        ▼
ITenantContext

Auditing
        │
        ▼
EF Core Interceptor
```

Bewerte diese Struktur und korrigiere sie, falls erforderlich.

---

# 21. Aggregate Repository

Entwickle ein Repository für Aggregate Roots.

Ausgangspunkt:

```csharp
public interface IRepository<TAggregate, TKey>
    where TAggregate : class, IAggregateRoot<TKey>
    where TKey : struct
{
    Task<TAggregate?> GetByIdAsync(
        TKey id,
        CancellationToken cancellationToken = default);

    Task AddAsync(
        TAggregate aggregate,
        CancellationToken cancellationToken = default);

    void Remove(TAggregate aggregate);
}
```

Bewerte:

- API
- Tracking
- `AsNoTracking`
- Cancellation Tokens
- Async
- Aggregate Loading
- Includes
- Specifications
- Transaktionen
- SaveChanges
- Unit of Work
- EF Core Leaks
- Testbarkeit

Vermeide unnötige CRUD-Abstraktionen.

---

# 22. Generic Repository vs. Concrete Repository

Vergleiche:

### A

```text
IRepository<TAggregate,TKey>
```

### B

```text
IOrderRepository
```

### C

```text
Generic Repository
        +
Concrete Repository
```

Bewerte:

- DDD
- Typsicherheit
- Wartbarkeit
- Query-Komplexität
- EF Core
- Testbarkeit
- Flexibilität
- Generic Repository Anti-Pattern

Treffe eine klare Empfehlung.

---

# 23. Concrete Aggregate Repository

Zeige mindestens:

```csharp
public interface IOrderRepository :
    IRepository<Order, OrderId>
{
}
```

und eine konkrete Implementierung.

Falls eine andere Architektur besser ist, verwende diese stattdessen und begründe sie.

Es darf kein Repository für:

```text
OrderLine
ShippingAddress
```

existieren.

---

# 24. Tenant Repository

Bewerte kritisch, ob folgende Abstraktion erforderlich ist:

```csharp
ITenantRepository<
    TAggregate,
    TKey,
    TTenantKey>
```

Vergleiche:

```text
Tenant Repository
```

mit:

```text
ITenantContext
+
EF Core Global Query Filters
```

sowie:

```text
Repository Decorator
```

und:

```text
PostgreSQL Row-Level Security
```

Treffe eine klare Entscheidung.

Wenn kein Tenant Repository erforderlich ist, sage ausdrücklich:

> Kein separates Tenant Repository.

und erkläre warum.

Die Single-Tenant-Variante darf diese Abstraktion überhaupt nicht benötigen.

---

# 25. Reference Data Repository

Entwickle ein separates Konzept für Reference Data.

Beispielsweise:

```csharp
IReferenceDataRepository<T,TKey>
```

Mögliche API:

```csharp
Task<T?> GetByIdAsync(...);

Task<IReadOnlyList<T>> GetAllAsync(...);
```

Prüfe, ob folgende Methoden bewusst fehlen sollten:

```text
AddAsync
UpdateAsync
DeleteAsync
```

Bei code-definierten Enum Entities sollen diese normalerweise nicht Bestandteil der API sein.

---

# 26. Cache Architecture

Trenne klar:

```text
Persistence
    ≠
Caching
```

Bewerte:

### Variante A

```text
ICacheRepository<T>
```

### Variante B

```text
IReferenceDataCache<T,TKey>
```

### Variante C

```text
CachedReferenceDataRepository
```

### Variante D

```text
ReferenceDataService
    ├── Cache
    └── Repository
```

Treffe eine klare Empfehlung.

Die Domain darf nichts über den Cache wissen.

---

# 27. Reference Data Service

Bewerte insbesondere folgende Struktur:

```text
Application
     │
     ▼
ReferenceDataService
     │
     ▼
ReferenceDataCache
     │
     │ Cache Miss
     ▼
ReferenceDataRepository
     │
     ▼
EF Core
     │
     ▼
PostgreSQL
```

Begründe, ob der Cache vor dem Repository liegen sollte oder als Repository-Decorator umgesetzt werden sollte.

---

# 28. Cache-aside

Implementiere bevorzugt ein nachvollziehbares Cache-aside-Verhalten.

Erster Zugriff:

```text
Application
    │
    ▼
Cache
    │
   Miss
    │
    ▼
Repository
    │
    ▼
PostgreSQL
    │
    ▼
Cache
    │
    ▼
Application
```

Weitere Zugriffe:

```text
Application
    │
    ▼
Cache
    │
   Hit
    │
    ▼
Application
```

Weitere Datenbankabfragen sollen vermieden werden, solange der Cache-Eintrag gültig ist.

---

# 29. Cache Stampede

Berücksichtige parallele Requests.

Problem:

```text
Request A ─┐
Request B ─┤
Request C ─┼── Cache Miss ──> mehrere DB Queries
Request D ─┤
Request E ─┘
```

Ziel:

```text
Request A ─┐
Request B ─┤
Request C ─┼── Cache Miss ──> 1 DB Query
Request D ─┤                      │
Request E ─┘                      ▼
                               Cache
                                  │
                 ┌────────────────┼───────────────┐
                 ▼                ▼               ▼
              Request A       Request B       Request C
```

Bewerte und implementiere einen geeigneten Mechanismus:

- `Lazy<Task<T>>`
- `SemaphoreSlim`
- `ConcurrentDictionary`
- Async Locks
- Single Flight

Berücksichtige:

- Race Conditions
- Deadlocks
- Fehlerbehandlung
- Cancellation
- Exception Caching
- Eviction
- Thread Safety
- parallele Zugriffe

Zeige echten C#-Code.

---

# 30. Cache-Technologie

Vergleiche:

```text
IMemoryCache
IDistributedCache
Redis
```

sowie:

```text
Cache-aside
Read-through
Write-through
```

Bewerte:

- Single Instance
- mehrere Application Instances
- Containerisierung
- Kubernetes
- horizontale Skalierung
- Latenz
- Speicher
- Ausfallsicherheit
- Cache Consistency
- Invalidierung
- Betriebskomplexität

Treffe eine Empfehlung für:

1. lokale Entwicklung
2. Single-Instance Production
3. Multi-Instance Production

---

# 31. PostgreSQL und Redis

Zeige klar:

```text
PostgreSQL
    =
Source of Truth
```

und:

```text
Redis / Memory Cache
    =
Performance Layer
```

Der Cache darf niemals als dauerhafte Primärquelle betrachtet werden.

Bewerte Ausfallszenarien:

```text
Cache unavailable
Database available
```

Die Anwendung soll bei einem Cache-Ausfall nicht automatisch ihre fachliche Datenbasis verlieren.

---

# 32. Cache Keys

Definiere eine konsistente Cache-Key-Strategie.

Single-Tenant:

```text
reference-data:OrderStatus
reference-data:PaymentType
reference-data:Country
```

Multi-Tenant nur dort, wo die Daten tatsächlich tenant-spezifisch sind:

```text
tenant:{tenantId}:reference-data:{type}
```

Entscheide ausdrücklich:

> Wann muss Tenant Bestandteil des Cache Keys sein und wann nicht?

Berücksichtige:

- Entity Type
- Key
- Tenant
- Environment
- Version
- Namespace
- Invalidation

Eine Single-Tenant-Anwendung soll keine künstliche Tenant-Komponente in jedem Cache Key benötigen.

---

# 33. Cache Invalidierung

Behandle:

- TTL
- Explicit Invalidation
- Refresh
- Deployment
- Migration
- Seed Data
- Application Restart
- Event-based Invalidation
- Distributed Invalidation

Bei code-definiertem Reference Data soll eine Änderung kontrolliert über Deployment/Migration erfolgen.

Definiere eine konkrete Strategie.

---

# 34. Lazy Loading vs. Startup Preloading

Vergleiche:

```text
Lazy Loading
```

```text
Startup Preloading
```

```text
Hybrid
```

Bewerte:

- Startup Time
- Memory
- First Request Latency
- Anzahl Reference-Data-Typen
- Fehlerbehandlung
- Multi-Instance Deployment

Treffe eine Empfehlung.

---

# 35. Mehrere Application Instances

Vergleiche:

```text
Database
   │
   ├── App A ── Memory Cache A
   ├── App B ── Memory Cache B
   └── App C ── Memory Cache C
```

mit:

```text
Database
   │
   ▼
Shared Redis
   │
   ├── App A
   ├── App B
   └── App C
```

Bewerte:

- Consistency
- Invalidierung
- Netzwerk-Latenz
- Failover
- Skalierung
- Betrieb

---

# 36. Unit of Work

EF Core `DbContext` ist bereits eine Unit-of-Work-artige Abstraktion.

Untersuche:

### Variante A

```text
Repository
    │
    ▼
DbContext
    │
    ▼
SaveChangesAsync()
```

### Variante B

```text
Repository
    │
    ▼
IUnitOfWork
    │
    ▼
DbContext
```

### Variante C

```text
Application Service
    │
    ├── Repository
    │
    └── UnitOfWork
```

Treffe eine klare Empfehlung.

Vermeide eine zusätzliche Abstraktion nur aus Gewohnheit.

Erkläre:

- Transaction Boundary
- SaveChanges
- mehrere Aggregate
- Domain Events
- Integration Events
- Outbox Pattern, falls relevant

---

# 37. Specification Pattern

Bewerte, ob Specifications sinnvoll sind.

Beispielsweise:

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>>? Criteria { get; }

    IReadOnlyCollection<
        Expression<Func<T, object>>> Includes { get; }
}
```

Berücksichtige:

- Filter
- Includes
- Sortierung
- Paging
- Tracking
- `AsNoTracking`
- Tenant Filter
- Projektionen

Vermeide:

```csharp
IQueryable<T>
```

als unkontrollierte öffentliche Repository-API.

Bewerte alternativ:

```csharp
Task<T?> FirstOrDefaultAsync(
    ISpecification<T> specification,
    CancellationToken cancellationToken = default);
```

und:

```csharp
Task<IReadOnlyList<T>> ListAsync(
    ISpecification<T> specification,
    CancellationToken cancellationToken = default);
```

Treffe eine klare Entscheidung.

---

# 38. CQRS

Bewerte CQRS.

Unterscheide:

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
DbContext
```

von:

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

Beantworte:

> Müssen alle Reads über Aggregate Repositorys laufen?

Bewerte:

- einfache CRUD Reads
- komplexe Queries
- Reporting
- Projektionen
- Dapper
- EF Core
- Performance
- Konsistenz

Treffe eine pragmatische Empfehlung statt CQRS dogmatisch überall einzusetzen.

---

# 39. PostgreSQL-spezifische Persistence

Da PostgreSQL der konkrete Datenbankprovider ist, untersuche explizit:

- Npgsql
- PostgreSQL `uuid`
- `integer`
- `bigint`
- Foreign Keys
- Composite Keys
- Unique Constraints
- Partial Indexes
- normale Indexes
- PostgreSQL `jsonb`
- Concurrency
- Transaktionen
- Row-Level Security
- Migrationen
- Seeding
- PostgreSQL-spezifische Constraints

`jsonb` darf nicht nur deshalb verwendet werden, weil PostgreSQL es unterstützt.

Für jedes PostgreSQL-spezifische Feature soll erklärt werden:

1. Warum ist es sinnvoll?
2. Welche Alternative gibt es?
3. Welche Auswirkungen hat es auf Portabilität?
4. Würdest du es in der finalen Architektur verwenden?

---

# 40. Concurrency

Untersuche Optimistic Concurrency für PostgreSQL + EF Core/Npgsql.

Bewerte mögliche Strategien:

- Concurrency Token
- Timestamp/Version
- PostgreSQL-spezifische Mechanismen
- `xmin`, falls sinnvoll
- explizite Versionsspalte

Treffe eine konkrete Empfehlung.

Zeige, wie ein Concurrency Conflict behandelt wird.

---

# 41. EF Core Configuration

Entwickle eine konsistente Configuration-Architektur.

Ausgangspunkt:

```csharp
public abstract class EntityConfiguration<T,TKey>
    : IEntityTypeConfiguration<T>
    where T : class, IEntity<TKey>
    where TKey : struct
{
}
```

Bewerte:

```text
EntityConfiguration
        │
        ├── AggregateRootConfiguration
        ├── ChildEntityConfiguration
        ├── EnumEntityConfiguration
        ├── Tenant Configuration
        └── Audit Configuration
```

Die Tenant- und Audit-Konfigurationen müssen optional sein.

Vermeide:

```text
TenantAuditableAggregateRootConfiguration
TenantAuditableChildEntityConfiguration
...
```

---

# 42. Configuration Composition

Bewerte:

- Vererbung
- Extension Methods
- Configuration Components
- Interfaces
- Helper Methods

Beispielsweise:

```text
OrderConfiguration
    │
    ├── Entity configuration
    ├── Aggregate configuration
    ├── optional Tenant configuration
    ├── optional Audit configuration
    └── Order-specific configuration
```

Treffe eine klare Entscheidung.

---

# 43. EF-Core Mapping

Zeige konkret:

```text
Order
  │
  ├── 1:N OrderLine
  ├── 1:0..1 ShippingAddress
  └── N:1 OrderStatus
```

Behandle:

- Foreign Keys
- Navigation Properties
- required / optional
- Delete Behavior
- Cascade Delete
- Owned Entities
- Owned Collections
- Value Objects
- Strongly Typed IDs
- Value Converters
- Backing Fields
- Aggregate Boundaries

---

# 44. Order Domain

Verwende mindestens:

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

IDs:

```text
OrderId
OrderLineId
TenantId
UserId
OrderStatusId
```

Reference Data:

```text
OrderStatus
PaymentType
Country
```

Eigenschaften:

- `Order` ist Aggregate Root.
- `OrderLine` ist Child Entity.
- `ShippingAddress` wird fachlich bewertet.
- `Order` kann tenant-aware sein.
- `Order` kann auditable sein.
- `OrderLine` kann auditierbar sein, wenn fachlich sinnvoll.
- `OrderLine` hat kein Repository.
- `ShippingAddress` hat kein Repository.
- `OrderRepository` arbeitet auf Aggregate-Ebene.
- `Order.StatusId` referenziert `OrderStatus`.
- `OrderStatus` wird nicht über normales CRUD gepflegt.
- Reference Data wird gecacht.

---

# 45. Repository-Matrix

Erstelle und korrigiere folgende Matrix:

| Typ | Repository | CRUD | Cache | Tenant |
|---|---|---:|---:|---:|
| Aggregate Root | Aggregate Repository | fachlich begrenzt | optional | optional |
| Child Entity | keines | Nein | Nein | über Aggregate |
| Enum Entity | Reference Data Repository | Nein | Ja | normalerweise Nein |
| Reference Data | Reference Data Repository | fachlich begrenzt | optional | optional |
| Value Object | keines | Nein | Nein | über Owner |

Begründe jede Zeile.

---

# 46. Dependency Injection

Untersuche:

```text
Singleton
Scoped
Transient
```

für:

```text
DbContext
Repository
ReferenceDataRepository
ReferenceDataService
Cache
TenantContext
CurrentUser
TimeProvider
Interceptors
```

Behandle insbesondere:

```text
DbContext = Scoped
Aggregate Repository = Scoped
```

Bewerte einen Singleton Memory Cache.

Bewerte Redis als Singleton bzw. connection-basierte Infrastructure.

Zeige konkrete `IServiceCollection`-Registrierungen.

Zeige explizit:

### Single-Tenant

```text
ITenantContext → nicht registriert
Tenant Resolution → nicht registriert
Tenant Repository → nicht registriert
Tenant-specific Cache Layer → nicht erforderlich
```

### Multi-Tenant

Nur tatsächlich benötigte Komponenten werden registriert.

---

# 47. Testbarkeit

Zeige Teststrategien für:

## Domain

```text
Order
OrderLine
Invariants
Equality
```

## Repository

```text
OrderRepository
ReferenceDataRepository
```

## Cache

Teste:

```text
Cache Hit
Cache Miss
Concurrent Cache Miss
Cache Invalidation
Cache Failure
```

Wichtiger Test:

```text
5 parallele Requests
        ↓
1 Database Query
        ↓
5 Cache Results
```

## Multi-Tenancy

```text
Tenant A cannot access Tenant B
```

## Enum Consistency

Teste:

```text
Enum ↔ Seed Data ↔ Database
```

## Integration

Verwende für Persistence-Tests eine echte PostgreSQL-Umgebung bzw. eine geeignete PostgreSQL-Teststrategie.

Erkläre, warum reine InMemory-Provider für bestimmte EF-Core-/PostgreSQL-Verhaltensweisen nicht ausreichend sind.

---

# 48. Security

Bewerte:

- Tenant Isolation
- Cross-Tenant Access
- Authorization
- Enum Manipulation
- Reference Data Manipulation
- Cache Poisoning
- falsche Cache Keys
- stale Tenant Context
- falsche DI Lifetime
- fehlende Database Constraints
- fehlende Foreign Keys
- Mass Assignment
- unkontrollierte Entity Updates

Unterscheide:

```text
Single-Tenant Security
```

und:

```text
Multi-Tenant Security
```

Zeige Defense-in-Depth:

```text
Application
    ↓
Repository / EF Core
    ↓
Database Constraints
    ↓
PostgreSQL Security Features
```

---

# 49. Vollständiger Beispielcode

Liefere für die finale Architektur vollständigen, untereinander konsistenten C#-Code.

## Domain

Nur tatsächlich benötigte Abstraktionen:

```text
IDomainObject
IEntity<TKey>
Entity<TKey>

IAggregateRoot<TKey>
AggregateRoot<TKey>

IChildEntity<TKey>
ChildEntity<TKey>

ITenantEntity<TTenantKey>
IAuditableEntity<TUserKey>

IReferenceDataEntity<TKey>
EnumEntity<...>
```

## IDs

```text
OrderId
OrderLineId
TenantId
UserId
OrderStatusId
```

## Domain

```text
Order
OrderLine
ShippingAddress
OrderStatus
PaymentType
Country
```

## Repository

```text
IRepository<TAggregate,TKey>
Repository<TAggregate,TKey>

IOrderRepository
OrderRepository

IReferenceDataRepository<T,TKey>
ReferenceDataRepository<T,TKey>
```

Nur falls architektonisch begründet:

```text
ITenantRepository
```

## Cache

```text
IReferenceDataCache<T,TKey>
ReferenceDataCache<T,TKey>
CacheKeyFactory
ReferenceDataService
```

einschließlich Concurrent-Cache-Miss-Schutz.

## EF Core

```text
AppDbContext

EntityConfiguration
AggregateRootConfiguration
ChildEntityConfiguration
EnumEntityConfiguration

OrderConfiguration
OrderLineConfiguration
ShippingAddressConfiguration
OrderStatusConfiguration
```

Tenant-/Audit-Konfiguration nur, wenn die finale Architektur sie tatsächlich benötigt.

## Audit

Zeige die finale Lösung, beispielsweise mit:

```text
SaveChangesInterceptor
ICurrentUser
TimeProvider
```

## Application

Zeige mindestens einen Command/Application Service, der:

1. `OrderRepository` verwendet
2. Reference Data über den vorgesehenen Zugriff bezieht
3. einen Domain Use Case ausführt
4. `SaveChangesAsync` korrekt behandelt
5. Cancellation Tokens verwendet

Zeige denselben Use Case für:

```text
Single-Tenant
```

und:

```text
Multi-Tenant
```

Der Single-Tenant-Code darf keine künstlichen Tenant-Abhängigkeiten enthalten.

---

# 50. PlantUML

Erstelle mehrere separate, syntaktisch korrekte und direkt kopierbare PlantUML-Diagramme.

Alle Diagramme müssen die Abstraktionshierarchie berücksichtigen:

```text
höchste Abstraktion
        ↓
Interfaces / abstrakte Konzepte
        ↓
konkrete Domain-Typen
        ↓
Persistence / Infrastructure
```

Nicht umgekehrt.

---

## 50.1 Entity Class Diagram

Darstellen:

```text
IEntity<TKey>
Entity<TKey>
IAggregateRoot<TKey>
AggregateRoot<TKey>
IChildEntity<TKey>
ChildEntity<TKey>
ITenantEntity<TTenantKey>
IAuditableEntity<TUserKey>
IReferenceDataEntity<TKey>
EnumEntity<...>
```

Zeige Generic Constraints über Notes oder Beschriftungen.

Tenant muss als optional erkennbar sein.

---

## 50.2 Aggregate Diagram

```text
Order
 ├── OrderLine
 ├── OrderLine
 └── ShippingAddress
```

Zeige:

- Aggregate Boundary
- Composition
- IDs
- Repository Boundary

---

## 50.3 Repository Diagram

```text
IOrderRepository
       │
       ▼
     Order
       │
       ├── OrderLine
       └── ShippingAddress
```

Es darf kein Child Repository erscheinen.

---

## 50.4 Reference Data / Enum Diagram

Zeige:

```text
C# Enum
   │
   │ numeric value
   ▼
Enum Entity
   │
   ├── Id
   ├── Name
   └── Metadata
   │
   ▼
PostgreSQL
```

sowie:

```text
Order.StatusId
       │
       ▼
OrderStatus.Id
```

---

## 50.5 Cache Diagram

```text
Application
    │
    ▼
ReferenceDataService
    │
    ▼
ReferenceDataCache
    │
 ┌──┴──┐
Hit   Miss
 │      │
 │      ▼
 │  ReferenceDataRepository
 │      │
 │      ▼
 │   EF Core
 │      │
 │      ▼
 │ PostgreSQL
 └──────┘
```

---

## 50.6 Cache Stampede Diagram

Zeige:

```text
5 Requests
    │
    ▼
5 DB Queries
```

gegen:

```text
5 Requests
    │
    ▼
1 coordinated DB Query
    │
    ▼
Cache
    │
    ▼
5 Responses
```

---

## 50.7 Multi-Instance Cache Diagram

Zeige:

```text
PostgreSQL
   │
   ├── App A ── Memory Cache A
   ├── App B ── Memory Cache B
   └── App C ── Memory Cache C
```

und:

```text
PostgreSQL
   │
   ▼
Redis
 ┌──┼──┐
 ▼  ▼  ▼
 A  B  C
```

---

## 50.8 EF Configuration Diagram

Zeige:

```text
EntityConfiguration
       │
       ├── Aggregate Configuration
       ├── Child Configuration
       ├── Enum Configuration
       ├── optional Tenant Configuration
       └── optional Audit Configuration
```

---

## 50.9 Single-Tenant vs. Multi-Tenant

Zeige:

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
EF Core
   │
   ▼
PostgreSQL
```

und:

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
Aggregate Repository / EF Core
   │
   ▼
Tenant Isolation
   │
   ▼
PostgreSQL
```

Das Diagramm muss klar zeigen:

> Multi-Tenancy erweitert die Basisarchitektur.

---

## 50.10 Gesamtarchitektur

Zeige:

```text
Application
   │
   ├── Commands
   │      │
   │      ▼
   │  Aggregate Repository
   │      │
   │      ▼
   │  Aggregate
   │      │
   │      ▼
   │   DbContext
   │      │
   │      ▼
   │  PostgreSQL
   │
   ├── Queries
   │      │
   │      ▼
   │  Query Handler
   │      │
   │      ├── EF Core
   │      └── Dapper
   │
   └── Reference Data
          │
          ▼
      ReferenceDataService
          │
          ▼
         Cache
          │
        Miss
          │
          ▼
      Repository
          │
          ▼
       EF Core
          │
          ▼
      PostgreSQL
```

Nur tatsächlich empfohlene Komponenten sollen im finalen Diagramm enthalten sein.

---

# 51. Abstraktions-Level in PlantUML

Ordne Klassen und Interfaces in den PlantUML-Diagrammen ausdrücklich nach Abstraktions-Level.

Grundregel:

```text
oben:
Interfaces
abstrakte Konzepte
generische Basistypen

↓

mittig:
konkrete Domain-Typen
Aggregate
Reference Data

↓

unten:
Repository Implementierungen
EF Core
Npgsql
PostgreSQL
Cache Infrastructure
```

Nutze beispielsweise PlantUML:

```plantuml
top to bottom direction
```

und geeignete `together`, Packages oder versteckte Beziehungen, wenn dadurch die gewünschte Anordnung stabiler wird.

Die Diagramme sollen nicht nur syntaktisch gültig, sondern auch visuell sinnvoll sein.

---

# 52. Architektur-Bewertung

Bewerte explizit:

1. Vererbung vs. Interfaces
2. Composition vs. Vererbung
3. Aggregate Root
4. Child Entity
5. Repository pro Aggregate
6. Generic Repository
7. Concrete Repository
8. Tenant Repository
9. Tenant Context
10. Global Query Filters
11. PostgreSQL Row-Level Security
12. Audit
13. SaveChanges Interceptor
14. Strongly Typed IDs
15. Entity Equality
16. EF-Core-Proxies
17. Persistierbare Enum Entities
18. C# Enum vs. Smart Enum
19. Enum Seed Data
20. numerischer Enum-Wert als Primary Key
21. String-Wert
22. zusätzliche Enum-Metadaten
23. Reference Data Repository
24. Cache Repository
25. Cache Decorator
26. ReferenceDataService
27. Cache-aside
28. Read-through
29. Lazy Loading
30. Startup Preloading
31. Cache Stampede
32. Cache Invalidierung
33. IMemoryCache
34. Distributed Cache
35. Redis
36. Repository vs. Query Handler
37. CQRS
38. Specification Pattern
39. Unit of Work
40. DbContext als Unit of Work
41. PostgreSQL
42. Npgsql
43. Concurrency
44. Testbarkeit
45. Performance
46. Skalierbarkeit
47. Multi-Instance Deployment
48. Single-Tenant Deployment
49. optionale Capabilities

Verwende für wesentliche Entscheidungen eine Struktur wie:

> **Empfehlung**

> **Alternative**

> **Warum**

> **Trade-off**

---

# 53. Finale Capability-Matrix

Erstelle am Ende eine korrigierte Matrix:

| Capability | Single-Tenant | Multi-Tenant |
|---|---|---|
| Entity | erforderlich | erforderlich |
| Aggregate Root | erforderlich | erforderlich |
| Child Entity | erforderlich | erforderlich |
| Aggregate Repository | erforderlich | erforderlich |
| Audit | optional | optional |
| Tenant Context | nicht erforderlich | erforderlich |
| Tenant Entity Interface | nicht erforderlich | nur für tenant-aware Entities |
| Tenant Query Filter | nicht erforderlich | erforderlich, sofern gewählt |
| Tenant Repository | möglichst nicht | nur bei konkreter Begründung |
| Reference Data Repository | optional | optional |
| Reference Data Cache | optional | optional |
| Redis | deploymentabhängig | deploymentabhängig |
| PostgreSQL RLS | nicht erforderlich | optional/empfohlen als Defense-in-Depth |
| CQRS | optional | optional |
| Specification | optional | optional |
| Unit of Work Interface | möglichst nicht | möglichst nicht |

Korrigiere die Tabelle entsprechend deiner finalen Architektur.

---

# 54. Finale Architekturentscheidung

Am Ende muss eine **eindeutige Zielarchitektur** stehen.

Beantworte explizit:

## Entities

- Wie sieht `Entity<TKey>` final aus?
- Gibt es Equality in der Basisklasse?
- Welche Interfaces existieren?
- Welche Generic Constraints existieren?

## Aggregate

- Wie wird ein Aggregate Root gekennzeichnet?
- Wie werden Child Entities modelliert?
- Wie wird Aggregate Encapsulation umgesetzt?
- Wie wird verhindert, dass Child Entities eigene Repositorys erhalten?

## Strongly Typed IDs

- Welche Implementierung wird verwendet?
- Wie werden sie in PostgreSQL gespeichert?
- Welche Value Converters werden benötigt?

## Tenant

- Ist Tenant Bestandteil der Basisarchitektur?
- Wie wird Tenant optional aktiviert?
- Gibt es ein Tenant Repository?
- Wo findet Tenant Resolution statt?
- Wo findet Tenant Filtering statt?
- Wo findet Tenant Authorization statt?
- Wo findet Tenant Isolation statt?
- Wird PostgreSQL RLS verwendet?

## Audit

- Interface oder Basisklasse?
- Wie werden Werte automatisch gesetzt?
- Welche Zeitquelle wird verwendet?
- Welche User-Abstraktion wird verwendet?

## Enum Entities

- Was ist eine Enum Entity?
- Wie wird die numerische Identität gespeichert?
- Wie wird der Name gespeichert?
- Wie werden Metadaten gespeichert?
- Wie wird Seed Data synchronisiert?
- Wie werden Änderungen erkannt?

## Repository

- Was ist das Aggregate Repository?
- Was ist das Reference Data Repository?
- Gibt es ein Tenant Repository?
- Gibt es ein Cache Repository?
- Wenn nein, warum nicht?

## Cache

- Welche Cache-Abstraktion wird verwendet?
- Cache-aside oder Decorator?
- Wie wird Concurrent Loading gelöst?
- Wie wird invalidiert?
- Wann wird Redis verwendet?
- Wie funktionieren mehrere Application Instances?
- Welche Cache Keys werden verwendet?

## EF Core / PostgreSQL

- Welche Configurations gibt es?
- Wie werden Strongly Typed IDs gemappt?
- Wie werden Aggregate und Child Entities gemappt?
- Wie werden Enum Entities gemappt?
- Wie werden Concurrency Tokens umgesetzt?
- Wie werden Tenant Query Filters umgesetzt?
- Wie wird Single-Tenant ohne Tenant Filters konfiguriert?

## CQRS

- Welche Reads laufen über Query Handler?
- Welche Reads laufen über Repositories?
- Wann ist Dapper sinnvoll?
- Wann reicht EF Core?

## Unit of Work

- Wird `IUnitOfWork` benötigt?
- Oder reicht `DbContext.SaveChangesAsync()`?

---

# 55. Architekturgrenzen

Die finale Architektur muss diese Richtung der Abhängigkeiten einhalten:

```text
Domain
   ↑
Application
   ↑
Infrastructure
```

beziehungsweise konzeptionell:

```text
Infrastructure
      │
      ▼
Application abstractions
      │
      ▼
Domain
```

Die Domain darf keine Abhängigkeit zu:

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

besitzen.

---

# 56. Textdiagramme

Zusätzlich zu PlantUML dürfen einfache Textdiagramme verwendet werden, wenn sie einen Ablauf besser darstellen.

Beispiel:

```text
Request
   │
   ▼
ReferenceDataService
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
   │
   ▼
Response
```

Verwende Textdiagramme insbesondere für:

- Cache-aside
- Cache Stampede
- Tenant Resolution
- Request Flow
- Command Flow
- Query Flow

---

# 57. Codequalität

Der finale C#-Code muss:

- syntaktisch plausibel sein
- konsistent sein
- moderne C#-Syntax verwenden, wenn sinnvoll
- nullable reference types berücksichtigen
- Cancellation Tokens verwenden
- Async korrekt einsetzen
- keine unnötigen Abstraktionen enthalten
- EF Core korrekt verwenden
- Npgsql/PostgreSQL berücksichtigen
- DI korrekt verwenden
- keine widersprüchlichen APIs enthalten

Vermeide Pseudocode, wenn echter C#-Code möglich ist.

Wenn Code aus Platzgründen vereinfacht wird, muss dies ausdrücklich gekennzeichnet werden.

Die einzelnen Codeblöcke müssen miteinander kompatibel sein.

---

# 58. Antwortstruktur

Strukturiere deine Antwort exakt in dieser Reihenfolge:

## 1. Executive Summary

Kurze Zusammenfassung der finalen Architektur.

## 2. Architekturprinzipien

Die wichtigsten Regeln.

## 3. Layer / Dependency Architecture

Domain, Application, Infrastructure.

## 4. Entity Model

Entity, Equality, Strongly Typed IDs.

## 5. Aggregate Architecture

Aggregate Root, Child Entity, Encapsulation.

## 6. Optional Capabilities

Audit, Multi-Tenancy, Caching.

## 7. Multi-Tenancy

Tenant Context, Resolution, Filtering, Isolation, RLS.

## 8. Audit

Audit Interfaces und Infrastructure.

## 9. Reference Data / Enum Entities

Enum Entity, Lookup Entity, Seed Data.

## 10. Repository Architecture

Aggregate Repository, Reference Data Repository, Tenant Repository.

## 11. Cache Architecture

Cache-aside, Concurrent Loading, Redis, Multi-Instance.

## 12. Unit of Work

DbContext vs. IUnitOfWork.

## 13. Specification / CQRS

Klare Empfehlung.

## 14. PostgreSQL / Npgsql

Provider-spezifische Entscheidungen.

## 15. EF Core Configuration

Configurations und Mapping.

## 16. Dependency Injection

Konkrete Registrierungen.

## 17. Beispielcode

Zusammenhängender C#-Code.

## 18. Tests

Domain, Integration, Repository, Cache, Tenant.

## 19. Security

Insbesondere Tenant Isolation.

## 20. PlantUML

Alle geforderten Diagramme.

## 21. Entscheidungs-Matrix

Alternativen und finale Entscheidung.

## 22. Finale Gesamtarchitektur

Eine kompakte Darstellung der tatsächlich empfohlenen Lösung.

## 23. Capability Matrix

Single-Tenant vs. Multi-Tenant.

## 24. Architekturregeln

Eine kurze Liste von verbindlichen Regeln für zukünftige Entwickler.

---

# 59. Entscheidungsregeln

Wenn mehrere Lösungen möglich sind:

1. Bevorzuge die einfachere Lösung.
2. Vermeide unnötige Abstraktionen.
3. Bevorzuge Composition gegenüber umfangreicher Vererbung.
4. Behandle Aggregate als fachliche Konsistenzgrenzen.
5. Repositorys sollen Aggregate-Grenzen respektieren.
6. Child Entities erhalten keine eigenen Repositorys.
7. Caching darf Persistence nicht ersetzen.
8. Multi-Tenancy darf Single-Tenancy nicht verkomplizieren.
9. Audit darf die Entity-Hierarchie nicht vervielfachen.
10. EF Core darf nicht in die Domain gelangen.
11. PostgreSQL soll dort genutzt werden, wo es einen echten Vorteil bietet.
12. Provider-spezifische Features sollen nicht ohne Begründung verwendet werden.
13. CQRS soll nur dort eingesetzt werden, wo es einen konkreten Nutzen bringt.
14. Ein `IUnitOfWork` soll nicht nur als Wrapper um `DbContext` eingeführt werden.
15. Ein Generic Repository darf nicht zu einer bloßen CRUD-Fassade über `IQueryable` werden.
16. Sicherheitsanforderungen dürfen nicht ausschließlich auf Application-Code vertrauen.
17. Tenant Isolation muss Defense-in-Depth berücksichtigen.
18. Cache Keys müssen Tenant-Grenzen korrekt berücksichtigen.
19. Code-definierte Enum-Werte müssen stabil bleiben.
20. Seed Data muss mit dem Code konsistent validiert werden.

---

# 60. Wichtigste finale Anforderung

Die finale Lösung muss diese Aussage erfüllen:

> **Multi-Tenancy erweitert die Architektur, anstatt die Basisarchitektur zu definieren.**

Das bedeutet:

### Single-Tenant

```text
Domain
  │
  ▼
Aggregate
  │
  ▼
Repository
  │
  ▼
EF Core / Npgsql
  │
  ▼
PostgreSQL
```

### Multi-Tenant

```text
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
Aggregate
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

Die beiden Architekturen sollen **dieselbe Domain- und Aggregate-Grundstruktur** verwenden.

Nur die zusätzlich erforderlichen Tenant-Komponenten werden aktiviert.

---

# 61. Wichtigste Abschlussfrage

Beende die Antwort mit einer kompakten, aber eindeutigen Zusammenfassung:

```text
Was ist die empfohlene Architektur?

Welche Interfaces gibt es?

Welche Basisklassen gibt es?

Welche Repositorys gibt es?

Welche Komponenten gibt es nur bei Multi-Tenancy?

Welche Komponenten gibt es nur für Audit?

Welche Komponenten gibt es nur für Caching?

Wie wird PostgreSQL eingebunden?

Wie wird EF Core eingebunden?

Wie wird Reference Data gespeichert?

Wie wird Reference Data gecacht?

Wie wird Tenant Isolation sichergestellt?

Wie bleibt Single-Tenancy einfach?

Welche Architekturentscheidungen sollten zukünftige Entwickler nicht ohne weiteres ändern?
```

Gib anschließend eine kompakte finale Architekturübersicht:

```text
                    ┌─────────────────────────┐
                    │         Domain          │
                    │                         │
                    │ Entity                  │
                    │ Aggregate Root          │
                    │ Child Entity            │
                    │ Value Objects           │
                    │ Domain Rules            │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │      Application        │
                    │                         │
                    │ Commands                │
                    │ Queries                 │
                    │ Repository Interfaces   │
                    │ Reference Data          │
                    │ optional Tenant         │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │    Infrastructure       │
                    │                         │
                    │ EF Core                 │
                    │ Npgsql                  │
                    │ PostgreSQL              │
                    │ Repositories            │
                    │ Cache                   │
                    │ Redis                   │
                    │ Interceptors            │
                    │ Tenant Infrastructure   │
                    └─────────────────────────┘
```

Dabei gilt als oberstes Architekturprinzip:

```text
                    BASE ARCHITECTURE
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
        Audit        Multi-Tenancy       Caching
       optional         optional         optional
```

Die Basisarchitektur muss vollständig ohne Multi-Tenancy funktionieren.

Die finale Empfehlung muss konkret, konsistent und produktionsnah sein und darf nicht lediglich eine Sammlung möglicher Optionen darstellen.