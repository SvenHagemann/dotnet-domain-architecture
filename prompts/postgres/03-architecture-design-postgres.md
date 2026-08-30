Ja. Da PostgreSQL jetzt **kein austauschbarer Nebenaspekt**, sondern der konkrete Persistence-Provider sein soll, würde ich den Prompt an einigen Stellen deutlich straffen und gleichzeitig präzisieren.

Insbesondere würde ich:
- die vielen Wiederholungen zu Multi-Tenancy, Cache und Repository zusammenführen,
- PostgreSQL/Npgsql als **verbindliche technische Grundlage** festlegen,
- PostgreSQL-spezifische Entscheidungen ausdrücklich verlangen,
- widersprüchliche Vorgaben wie „generisches Repository“ vs. „nicht automatisch Generic Repository“ sauber formulieren,
- die erwarteten Deliverables am Ende zentral definieren,
- und vermeiden, dass das nachfolgende Modell 56 einzelne Abschnitte abarbeitet, ohne eine konsistente Architektur zu entwickeln.

Der aktuelle Stand von EF Core führt `Npgsql.EntityFrameworkCore.PostgreSQL` als PostgreSQL-Provider; die Provider-Version muss zur verwendeten EF-Core-Hauptversion passen. PostgreSQL wird über `UseNpgsql(...)` konfiguriert. 

Hier ist die **bereinigte und konsolidierte Version**, die ich für die eigentliche Architekturarbeit verwenden würde:

:::writing{variant="document" id="58321"}
# Aufgabe

Entwirf eine **robuste, praxisnahe, konsistente und langfristig wartbare Gesamtarchitektur** für eine verteilte E-Commerce-Webapplikation auf Basis von:

- C#
- ASP.NET Core
- Entity Framework Core
- PostgreSQL
- Npgsql.EntityFrameworkCore.PostgreSQL
- Dependency Injection
- optional CQRS
- optional Dapper für spezialisierte Read Queries

Die Architektur soll Domain- und Persistence-Aspekte sauber trennen und anschließend durch mehrere **direkt kopierbare PlantUML-Diagramme** visualisiert werden.

Die konkrete Anwendung umfasst unter anderem die Services:

```text
Basket
Catalog
Discount
Ordering
Payment
Auth
```

Für die konkrete Demonstration soll eine kleine **Order-Domain** verwendet werden:

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

mit:

```text
OrderId
OrderLineId
optional TenantId
UserId
OrderStatusId
```

---

# 1. Zentrale Architekturziele

Die Architektur muss folgende Eigenschaften erfüllen:

1. **Domain-First**
   - Die Domain soll möglichst wenig technische Abhängigkeiten besitzen.
   - EF Core, PostgreSQL, Redis und ASP.NET Core sollen nicht Bestandteil der Domain sein.

2. **Aggregate-orientiert**
   - Aggregate Roots bilden die fachlichen Repository-Grenzen.
   - Child Entities werden ausschließlich über ihr Aggregate verwaltet.
   - Child Entities erhalten keine eigenen Repositories.

3. **Strongly Typed IDs**
   - IDs wie `OrderId`, `CustomerId`, `TenantId` und `UserId` müssen typsicher voneinander getrennt sein.

4. **Optionale Capabilities**
   - Multi-Tenancy
   - Auditing
   - Caching
   - CQRS

   dürfen die Basisarchitektur nicht unnötig verkomplizieren.

5. **PostgreSQL als verbindlicher Persistence Provider**
   - EF Core soll mit Npgsql für PostgreSQL verwendet werden.
   - PostgreSQL-spezifische Möglichkeiten sollen berücksichtigt werden, wenn sie einen echten architektonischen Vorteil bieten.
   - Provider-spezifische Lösungen sollen aber nicht unnötig in Domain oder Application Layer gelangen.

6. **Produktionsgeeignet**
   - Thread Safety
   - Concurrency
   - Transaktionen
   - Tenant Isolation
   - Cache Consistency
   - Testbarkeit
   - Skalierbarkeit
   - Multi-Instance Deployment
   - Migrationen
   - Seed Data
   - Fehlerbehandlung
   - DI Lifetimes

   müssen berücksichtigt werden.

---

# 2. Wichtigste architektonische Leitlinie: Basisarchitektur und optionale Capabilities

Die Basisarchitektur soll vollständig ohne Multi-Tenancy funktionieren.

Konzeptionell:

```text
                    ┌────────────────────┐
                    │  Basisarchitektur  │
                    │                    │
                    │ Entity             │
                    │ Aggregate          │
                    │ Child Entity       │
                    │ Repository         │
                    │ EF Core            │
                    │ PostgreSQL         │
                    └──────────┬─────────┘
                               │
                    optionale Capabilities
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
            Audit         Multi-Tenancy       Cache
```

Die Architektur darf nicht so aufgebaut werden, dass beispielsweise jede Entity automatisch:

```text
TenantId
ITenantContext
Tenant Query Filter
Tenant Repository
Tenant-aware Cache
```

benötigt.

Eine einfache Single-Tenant-Entity soll beispielsweise weiterhin möglich sein:

```csharp
public sealed class Product
    : AggregateRoot<ProductId>
{
}
```

Eine Tenant-fähige Entity kann dagegen explizit Tenant-aware sein:

```csharp
public sealed class Order
    : AggregateRoot<OrderId>,
      ITenantEntity<TenantId>
{
}
```

Bewerte diese Lösung kritisch und entwickle sie gegebenenfalls weiter.

---

# 3. Architektur-Schichten

Entwickle eine klare Schichtung:

```text
┌──────────────────────────────────────────┐
│                 Domain                   │
│                                          │
│ Entities                                 │
│ Aggregate Roots                          │
│ Child Entities                           │
│ Value Objects                            │
│ Domain Rules                             │
│ Domain Interfaces                        │
│ Strongly Typed IDs                       │
└────────────────────┬─────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────┐
│               Application                │
│                                          │
│ Use Cases                                │
│ Commands                                 │
│ Queries                                  │
│ Application Services                     │
│ Repository Interfaces                    │
│ Reference Data Interfaces                │
│ Cache Interfaces                         │
└────────────────────┬─────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────┐
│             Infrastructure               │
│                                          │
│ EF Core                                  │
│ Npgsql / PostgreSQL                      │
│ Repository Implementations              │
│ Cache Implementations                    │
│ Redis                                    │
│ Audit Interceptors                       │
│ Tenant Infrastructure                    │
│ Dapper                                   │
└──────────────────────────────────────────┘
```

Begründe, welche Abstraktion in welcher Schicht liegt.

---

# 4. Entity-Modell

Bewerte kritisch folgende Ausgangsstruktur:

```csharp
public interface IEntity<TKey> : IDomainObject
    where TKey : struct
{
    TKey Id { get; }
}

public abstract class Entity<TKey> :
    IEntity<TKey>
    where TKey : struct
{
    public TKey Id { get; protected init; }
}
```

Entwickle daraus die finale Entity-Abstraktion.

Untersuche insbesondere:

- Entity Equality
- HashCode
- transient entities
- Strongly Typed IDs
- polymorphe Entities
- Vererbung
- EF-Core-Proxies
- Tracking
- Collections
- Dictionary Keys
- wechselnde Proxy-Typen
- mutable IDs
- `init` vs. `set`
- Domain vs. Persistence Concerns

Bewerte auch die ursprünglich vorgeschlagene Equality-Implementierung:

```csharp
public override bool Equals(object? obj)
{
    if (obj == null || obj is not Entity<TKey>)
        return false;

    return Equals((Entity<TKey>)obj);
}
```

und insbesondere die darin enthaltene Typprüfung:

```csharp
if (!thisType.IsAssignableFrom(otherType) &&
    !otherType.IsAssignableFrom(thisType))
    return false;
```

Entscheide, ob diese Equality-Strategie für eine produktive EF-Core-Anwendung geeignet ist.

Wenn Änderungen erforderlich sind:

1. liefere die finale Implementierung,
2. erkläre das Problem,
3. erkläre die Auswirkungen auf EF Core,
4. erkläre die Auswirkungen auf Collections und Hashing.

---

# 5. Strongly Typed IDs

Verwende Strongly Typed IDs, beispielsweise:

```csharp
public readonly record struct OrderId(Guid Value);

public readonly record struct OrderLineId(Guid Value);

public readonly record struct TenantId(Guid Value);

public readonly record struct UserId(Guid Value);

public readonly record struct OrderStatusId(int Value);
```

Untersuche:

- Generic Constraints
- `where TKey : struct`
- Value Objects
- EF-Core Value Converters
- PostgreSQL-Datentypen
- Primary Keys
- Foreign Keys
- Composite Keys
- Performance
- Typsicherheit

Zeige konkret, wie verhindert wird, dass beispielsweise:

```csharp
CustomerId
```

an eine Methode übergeben wird, die:

```csharp
OrderId
```

erwartet.

Berücksichtige PostgreSQL-spezifisch die sinnvolle Abbildung von:

```text
Guid
int
string
DateTime / timestamp
```

und entscheide, welche PostgreSQL-Typen verwendet werden sollen.

---

# 6. Aggregate Roots

Entwickle eine finale Struktur für Aggregate Roots.

Beispielsweise:

```csharp
public interface IAggregateRoot<TKey> : IEntity<TKey>
    where TKey : struct
{
}
```

Bewerte kritisch, ob zusätzlich:

```csharp
public abstract class AggregateRoot<TKey>
    : Entity<TKey>,
      IAggregateRoot<TKey>
    where TKey : struct
{
}
```

sinnvoll ist.

Erkläre:

- Unterschied zwischen Entity und Aggregate Root
- Aggregate Boundary
- Repository Boundary
- Aggregate Invariants
- Type-System-Unterstützung
- Grenzen von Generic Constraints

---

# 7. Child Entities

Verwende für die Order-Domain:

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

`Order` ist Aggregate Root.

`OrderLine` gehört ausschließlich zu `Order`.

`ShippingAddress` soll kritisch als:

```text
Child Entity
```

versus:

```text
Value Object / Owned Entity
```

bewertet werden.

Treffe anschließend eine klare Entscheidung.

Child Entities dürfen keine eigenen Repositories besitzen.

Insbesondere darf es nicht geben:

```csharp
IRepository<OrderLine, OrderLineId>
```

Erkläre, wie die Architektur dies durch Aggregate Boundaries und Typisierung unterstützt und welche Grenzen das C#-Typsystem besitzt.

---

# 8. Aggregate Encapsulation

Das Aggregate soll seine Child Entities selbst verwalten.

Beispielsweise:

```csharp
order.AddLine(...);
order.RemoveLine(...);
order.ChangeShippingAddress(...);
```

und nicht:

```csharp
order.OrderLines.Add(...);
order.OrderLines.Remove(...);
```

Zeige eine geeignete Implementierung mit:

```csharp
private readonly List<OrderLine> _lines = [];

public IReadOnlyCollection<OrderLine> Lines =>
    _lines;
```

Berücksichtige:

- Invarianten
- Child Lifecycle
- Navigation Properties
- EF-Core-Kompatibilität
- Encapsulation
- Domain Methods

---

# 9. Optionale Multi-Tenancy

Multi-Tenancy ist eine **optionale Capability**.

Single-Tenant ist der Basisfall.

Untersuche insbesondere:

```csharp
public interface ITenantEntity<TTenantKey>
    where TTenantKey : struct
{
    TTenantKey TenantId { get; }
}
```

sowie Alternativen:

- Interface
- Basisklasse
- Composition
- Generic Constraints
- Tenant Context
- Repository Decorator
- Application Service
- EF Core Global Query Filter
- EF Core Interceptor
- PostgreSQL Row-Level Security

Vermeide eine Pflicht-Basisklasse wie:

```text
TenantEntity<TKey,TTenantKey>
```

für sämtliche Entities.

---

# 10. Tenant-Verantwortlichkeiten

Unterscheide ausdrücklich:

```text
Tenant Resolution
Tenant Context
Tenant Filtering
Tenant Authorization
Tenant Data Isolation
```

und ordne jede Verantwortung einer geeigneten Schicht zu.

Untersuche:

```text
ITenantContext
```

als Request-/Application-Context.

Bewerte, ob ein:

```text
ITenantRepository<TAggregate,TKey,TTenantKey>
```

notwendig ist.

Treffe eine klare Entscheidung.

Eine bevorzugte Lösung könnte beispielsweise sein:

```text
Request
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
EF Core Query Filter
   │
   ▼
PostgreSQL
```

Dies ist jedoch nur ein Ausgangspunkt.

---

# 11. Tenant Isolation und PostgreSQL

Tenant Isolation ist ein **Security Requirement**.

Untersuche mindestens:

### Anwendungsebene

```text
Tenant Resolution
Tenant Authorization
```

### EF-Core-Ebene

```text
Global Query Filters
SaveChanges Validation
Interceptors
```

### Datenbankebene

```text
Foreign Keys
Unique Constraints
Indexes
Composite Keys
PostgreSQL Row-Level Security
```

Bewerte insbesondere PostgreSQL Row-Level Security (RLS).

Erkläre, ob RLS:

- zusätzlich zu Application/EF-Core-Isolation,
- statt EF-Core-Isolation,
- oder nur in besonders sicherheitskritischen Szenarien

eingesetzt werden sollte.

Untersuche außerdem:

```text
PK = EntityId
```

versus:

```text
PK = TenantId + EntityId
```

und entscheide begründet.

Berücksichtige Foreign Keys, Unique Constraints und Index-Design.

---

# 12. Audit

Audit ist ebenfalls optional.

Entwickle eine geeignete Struktur für:

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
- EF Core SaveChanges
- SaveChangesInterceptor
- Current User Context
- Time Provider
- Domain vs. Infrastructure

Vermeide Klassenexplosion wie:

```text
AuditableEntity
TenantEntity
AuditableTenantEntity
TenantAggregateRoot
AuditableTenantAggregateRoot
...
```

Treffe eine klare Entscheidung.

---

# 13. Persistierbare typisierte Enum Entities

Der Begriff **Enum Entity** meint ausdrücklich kein normales CRUD-Lookup.

Es soll ein durch Code definierter, persistierbarer, typisierter Enum-Wertebereich entstehen.

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

Die persistierte Struktur soll beispielsweise enthalten:

```text
OrderStatus
 ├── Id
 ├── Name
 ├── IsFinal
 ├── AllowsModification
 ├── SortOrder
 └── weitere Metadaten
```

Dabei gilt:

- `Id` entspricht dem numerischen Enum-Wert.
- `Id` ist stabil.
- `Id` ist Primary Key.
- `Id` wird nicht von PostgreSQL generiert.
- Der Name wird explizit persistiert.
- Weitere Metadaten sind möglich.
- Benutzer dürfen diese Werte nicht über normales CRUD verwalten.
- Andere Entities dürfen den numerischen Wert referenzieren.
- Die Werte werden durch Code definiert.
- Seed Data muss mit dem Code konsistent bleiben.

---

# 14. Enum Entity vs. Lookup Entity

Erkläre den Unterschied zwischen:

```text
Lookup Entity
```

und:

```text
persistierbarer Enum Entity
```

Beispiel Lookup:

```text
Country
CustomerCategory
PaymentMethod
```

Beispiel Enum Entity:

```text
OrderStatus
Pending
Confirmed
Shipped
Cancelled
```

Erkläre, warum eine Enum Entity kein normales:

```text
Add
Update
Delete
```

CRUD benötigt.

---

# 15. Enum-Implementierung

Bewerte mindestens:

### Variante A

```text
C# enum + persistierte Entity
```

### Variante B

```text
Smart Enum / Enumeration Pattern
```

### Variante C

```text
generische EnumEntity<TKey,TEnum>
```

Entwickle gegebenenfalls eine bessere Variante.

Bewerte:

- Typsicherheit
- numerische Identität
- Name
- zusätzliche Metadaten
- EF Core
- PostgreSQL
- Seed Data
- Migrationen
- Testbarkeit
- Cache
- Performance

Entscheide dich anschließend für eine Variante.

---

# 16. Enum ↔ PostgreSQL-Konsistenz

Definiere eine robuste Strategie für:

```text
C# Enum
       ↕
Seed Data
       ↕
PostgreSQL
```

Bewerte:

- `HasData`
- Migration Seed
- Custom Seeder
- Startup Validation
- Integration Tests
- Build-Time Validation

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

Definiere, wie solche Inkonsistenzen erkannt und behandelt werden.

---

# 17. Repository-Architektur

Entwickle eine konsistente Repository-Architektur.

Die wichtigste Frage lautet:

> Welche Abstraktionen sind tatsächlich Repositorys und welche sollten besser als Service, Context, Decorator oder Infrastructure Component modelliert werden?

Bewerte insbesondere:

```text
Aggregate Repository
Reference Data Repository
Tenant Repository
Cache Repository
Unit of Work
Tenant Context
Cache
Query Handler
```

Vermeide eine künstliche Hierarchie wie:

```text
IRepository
 ├── ICacheRepository
 ├── ITenantRepository
 ├── IAuditRepository
 ├── IEnumRepository
 ├── IAggregateRepository
 └── IChildEntityRepository
```

wenn diese unterschiedliche Verantwortlichkeiten vermischt.

---

# 18. Aggregate Repository

Untersuche:

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

Entwickle daraus die finale Lösung.

Das Repository muss:

- ausschließlich Aggregate Roots akzeptieren,
- Child Entities ausschließen,
- Aggregate Loading unterstützen,
- Tracking sinnvoll behandeln,
- Cancellation Tokens unterstützen,
- EF Core verwenden können,
- keine Domain-Infrastrukturabhängigkeit erzeugen,
- keine unnötigen Transaktionen verstecken.

Bewerte kritisch:

```text
Generic Repository
```

gegen:

```text
Concrete Repository
```

und:

```text
Generic Base Repository + Concrete Repository
```

Treffe eine klare Empfehlung.

---

# 19. Concrete Repository

Zeige für `Order`:

```csharp
public interface IOrderRepository
    : IRepository<Order, OrderId>
{
}
```

und eine produktionsnahe Implementierung.

Falls ein generisches Repository als Base Class nicht empfohlen wird, zeige stattdessen die bessere Lösung.

---

# 20. Reference Data Repository

Entwickle ein eigenes Konzept für:

```text
Enum Entities
Reference Data
Stammdaten
```

beispielsweise:

```csharp
IReferenceDataRepository<T,TKey>
```

Das Repository soll nicht automatisch vollständiges CRUD anbieten.

Typische Operationen:

```csharp
Task<T?> GetByIdAsync(...);

Task<IReadOnlyList<T>> GetAllAsync(...);
```

Bewerte, welche Operationen tatsächlich notwendig sind.

---

# 21. Cache-Architektur

Trenne strikt:

```text
Persistence
    ≠
Caching
```

Ein Cache ist kein Repository und kein Datenbankersatz.

Bewerte:

```text
IReferenceDataCache<T,TKey>
CachedReferenceDataRepository<T,TKey>
ReferenceDataService
```

und entscheide dich für eine Architektur.

Eine mögliche Zielstruktur:

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

---

# 22. Cache-aside und Concurrent Cache Misses

Der Cache soll nach dem Cache-aside-Prinzip funktionieren:

```text
Request
   │
   ▼
 Cache
 ┌─┴─┐
Hit Miss
 │   │
 │   ▼
 │ Database
 │   │
 └───┘
   │
   ▼
Response
```

Beim ersten Zugriff soll die Datenbank geladen werden.

Weitere Requests sollen aus dem Cache bedient werden.

Verhindere Cache Stampede:

```text
5 Requests
    │
    ▼
5 DB Queries
```

stattdessen:

```text
5 Requests
    │
    ▼
1 DB Query
    │
    ▼
Cache
    │
    ▼
5 Results
```

Bewerte:

- `Lazy<Task<T>>`
- `SemaphoreSlim`
- `ConcurrentDictionary`
- Async Locks
- Single Flight

und implementiere einen geeigneten Mechanismus.

Berücksichtige:

- Thread Safety
- Race Conditions
- Deadlocks
- Cancellation
- Fehler beim Laden
- globale Locks
- Memory Usage

---

# 23. Cache-Technologie

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

Berücksichtige:

- Single Instance
- mehrere Application Instances
- Containerisierung
- horizontale Skalierung
- Latenz
- Ausfallsicherheit
- Speicherverbrauch
- Cache Consistency
- Invalidation

Treffe eine klare Empfehlung für:

```text
Single Instance
```

und:

```text
Multi-Instance Deployment
```

---

# 24. Cache Keys und Multi-Tenancy

Definiere eine konsistente Cache-Key-Strategie.

Single-Tenant beispielsweise:

```text
reference-data:OrderStatus
reference-data:PaymentType
reference-data:Country
```

Multi-Tenant gegebenenfalls:

```text
tenant:{tenantId}:reference-data:{type}
```

Entscheide, welche Reference Data tenant-spezifisch sein darf.

Ein Tenant darf niemals Daten eines anderen Tenants aus dem Cache erhalten.

Berücksichtige:

- Namespace
- Type
- Key
- optional Tenant
- Version
- Environment
- Invalidation

---

# 25. Cache Invalidation

Behandle:

- TTL
- Explicit Invalidation
- Refresh
- Application Restart
- Deployment
- Migration
- Seed Data
- Event-based Invalidation
- Distributed Invalidation

Erkläre insbesondere, wie sich die Strategie bei:

```text
IMemoryCache
```

und:

```text
Redis
```

unterscheidet.

---

# 26. PostgreSQL und Caching

Die Datenbank ist:

```text
PostgreSQL
```

Der Cache ist eine davon getrennte Infrastrukturkomponente.

Zeige die Beziehung:

```text
ReferenceDataService
       │
       ▼
     Cache
       │
     Miss
       │
       ▼
ReferenceDataRepository
       │
       ▼
   EF Core/Npgsql
       │
       ▼
   PostgreSQL
```

Bewerte, ob PostgreSQL-spezifische Mechanismen wie Notifications/Events für Cache-Invalidierung sinnvoll sind oder ob eine explizite Application-/Distributed-Cache-Strategie vorzuziehen ist.

---

# 27. Unit of Work

EF Core `DbContext` besitzt bereits wesentliche Unit-of-Work-Eigenschaften.

Bewerte deshalb kritisch, ob:

```csharp
IUnitOfWork
```

notwendig ist.

Vergleiche:

```text
Application Service
      │
      ▼
Repository
      │
      ▼
DbContext
      │
      ▼
SaveChangesAsync()
```

mit:

```text
Application Service
      │
      ├── Repository
      │
      └── IUnitOfWork
             │
             ▼
          DbContext
```

Treffe eine klare Empfehlung.

---

# 28. Transactions

Erkläre:

- wann EF Core `SaveChangesAsync()` genügt,
- wann explizite Transaktionen erforderlich sind,
- welche Rolle PostgreSQL-Transaktionen spielen,
- wann mehrere Aggregate in einer Transaktion verändert werden dürfen,
- wann ein Distributed Transaction vermieden werden sollte.

Berücksichtige die verteilte E-Commerce-Architektur und erkläre, wie Transaktionen über Service-Grenzen behandelt werden sollten.

---

# 29. Specification Pattern

Bewerte kritisch, ob ein Specification Pattern notwendig ist.

Untersuche:

```text
Criteria
Includes
Sorting
Paging
Tracking
AsNoTracking
Projection
Tenant Filter
```

Vermeide, dass Repositorys einfach:

```csharp
IQueryable<T>
```

nach außen geben.

Falls Specification empfohlen wird, entwickle eine konkrete, EF-Core-kompatible API.

---

# 30. CQRS

Bewerte CQRS für die Architektur.

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
   │
   ▼
PostgreSQL
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

Erkläre, warum Queries nicht zwangsläufig über Aggregate Repositories laufen müssen.

Treffe eine klare Empfehlung, ob:

```text
CQRS light
```

für diese Architektur sinnvoll ist.

---

# 31. EF Core und PostgreSQL

Entwickle die konkrete Persistence-Architektur mit:

```text
EF Core
   │
   ▼
Npgsql
   │
   ▼
PostgreSQL
```

Zeige:

```csharp
AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(connectionString);
});
```

Berücksichtige die zum verwendeten EF-Core-/Npgsql-Release passende Konfiguration.

Bewerte insbesondere:

- PostgreSQL UUID
- Integer
- timestamp / timestamptz
- varchar/text
- numeric
- JSONB, falls sinnvoll
- PostgreSQL-Indizes
- Foreign Keys
- Unique Constraints
- Composite Indexes
- Concurrency
- Migrations
- Connection Pooling

Vermeide PostgreSQL-spezifische Details in der Domain.

---

# 32. EF-Core Configuration

Entwickle eine wartbare Configuration-Architektur.

Bewerte beispielsweise:

```text
EntityConfiguration<T,TKey>
       │
       ├── Aggregate Configuration
       ├── Child Configuration
       ├── Enum Entity Configuration
       ├── Tenant Configuration
       └── Audit Configuration
```

Vermeide Klassenexplosion.

Bevorzugte Alternativen können sein:

- Helper Methods
- Extension Methods
- Configuration Components
- Interfaces
- begrenzte Vererbung

Treffe eine klare Entscheidung.

Tenant- und Audit-Konfiguration dürfen nur auf Entities angewendet werden, die die jeweilige Capability tatsächlich implementieren.

---

# 33. EF-Core Beziehungen

Zeige für die Order-Domain:

```text
Order
  │
  ├── 1:N OrderLine
  ├── 1:0..1 ShippingAddress
  └── N:1 OrderStatus
```

Erkläre:

- Primary Keys
- Foreign Keys
- Required / Optional
- DeleteBehavior
- Cascade Delete
- Value Converters
- Owned Entities
- Navigation Properties
- Aggregate Boundary

Entscheide insbesondere über:

```text
ShippingAddress = Child Entity
```

oder:

```text
ShippingAddress = Value Object / Owned Entity
```

---

# 34. Enum Entity Persistence

Zeige konkret die PostgreSQL-Struktur:

```text
order_status
----------------------------
id
name
is_final
allows_modification
sort_order
...
```

mit:

```text
id = numerischer Enum-Wert
```

Beispielsweise:

```text
1 | Pending
2 | Confirmed
3 | Shipped
4 | Cancelled
```

EF Core muss sicherstellen:

```csharp
.ValueGeneratedNever()
```

für die Enum-ID.

Zeige die Seed-Strategie.

---

# 35. Order-Domain

Implementiere eine konkrete Order-Domain.

Mindestens:

```text
Order
OrderLine
ShippingAddress
OrderStatus
PaymentType
Country
```

Dabei gilt:

```text
Order = Aggregate Root
OrderLine = Child Entity
ShippingAddress = nach Bewertung Entity oder Value Object
OrderStatus = persistierbare Enum Entity
PaymentType = Reference Data
Country = Reference Data
```

`OrderStatus` besitzt:

```text
Id
Name
IsFinal
AllowsModification
SortOrder
```

und wird nicht über normales CRUD verwaltet.

---

# 36. Beispiel für Single-Tenant und Multi-Tenant

Zeige dieselbe Order-Domain in:

```text
Single-Tenant
```

und:

```text
Multi-Tenant
```

Die Unterschiede sollen minimal sein.

Single-Tenant:

```text
Order
 └── kein Tenant erforderlich
```

Multi-Tenant:

```text
Order
 └── ITenantEntity<TenantId>
```

Zeige insbesondere, dass Application Services und Repositorys nicht unnötig mit Tenant-Generics überladen werden.

---

# 37. Dependency Injection

Zeige konkrete DI-Registrierungen für:

```text
DbContext
Repositories
ReferenceDataRepository
ReferenceDataService
Cache
TenantContext
Audit Services
Query Handlers
```

Berücksichtige:

```text
Scoped
Singleton
Transient
```

und entscheide jeweils begründet.

Typischer Ausgangspunkt:

```text
DbContext       Scoped
Repository      Scoped
Application     Scoped
TenantContext   Scoped
Cache           Singleton
```

Der Cache muss thread-safe sein.

Für Redis/Distributed Cache muss die entsprechende Lifetime- und Abstraktionsentscheidung erläutert werden.

Zeige ausdrücklich, welche Services in:

```text
Single-Tenant
```

nicht registriert werden müssen.

---

# 38. Audit Implementation

Implementiere eine produktionsnahe Lösung mit beispielsweise:

```text
SaveChangesInterceptor
ICurrentUser
TimeProvider
```

Zeige:

```text
CreatedAt
CreatedBy
ModifiedAt
ModifiedBy
```

und erkläre, warum diese Logik nicht in die Domain-Basisklasse eingebaut werden sollte, sofern dies die bessere Lösung ist.

---

# 39. Testbarkeit

Definiere eine Teststrategie für:

### Domain

```text
Order
OrderLine
Aggregate Invariants
Equality
Strongly Typed IDs
```

### Persistence

```text
OrderRepository
ReferenceDataRepository
EF Core mappings
PostgreSQL integration
```

### Enum Data

```text
Enum ↔ Database consistency
Seed Data
```

### Cache

```text
Hit
Miss
Concurrent Miss
Invalidation
Expiration
```

Insbesondere:

```text
5 concurrent requests
        ↓
1 database query
        ↓
5 results
```

### Multi-Tenancy

Nur bei aktivierter Multi-Tenancy:

```text
Tenant A cannot access Tenant B
```

### Integration

Verwende für Persistence-Tests bevorzugt eine echte PostgreSQL-Instanz bzw. einen PostgreSQL-basierten Testansatz und nicht ausschließlich EF Core InMemory, wenn reale relationale/provider-spezifische Semantik geprüft werden soll.

---

# 40. Security

Berücksichtige:

- Tenant Isolation
- Cross-Tenant Access
- falsche Tenant Contexts
- Cache Key Isolation
- Cache Poisoning
- manipulierte Enum IDs
- manipulierte Reference Data
- fehlende Foreign Keys
- fehlende Unique Constraints
- falsche DI Lifetimes

Unterscheide:

```text
Single-Tenant Security
```

von:

```text
Multi-Tenant Security
```

und erkläre die Verantwortung von:

```text
Application
EF Core
PostgreSQL
```

---

# 41. Vollständiger Beispielcode

Liefere für die finale Architektur **konsistenten, möglichst produktionsnahen C#-Code**.

Mindestens:

## Domain

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

Nur Interfaces/Basisklassen, die tatsächlich in der finalen Architektur benötigt werden.

## IDs

```text
OrderId
OrderLineId
TenantId
UserId
OrderStatusId
PaymentTypeId
CountryId
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
Repository<TAggregate,TKey>       // falls empfohlen

IOrderRepository
OrderRepository

IReferenceDataRepository<T,TKey>
ReferenceDataRepository<T,TKey>
```

Ein `ITenantRepository` nur dann, wenn es nach der Bewertung tatsächlich notwendig ist.

## Cache

```text
IReferenceDataCache<T,TKey>
ReferenceDataCache<T,TKey>
CacheKeyFactory
ReferenceDataService
```

inklusive Schutz vor Concurrent Cache Misses.

## EF Core

```text
AppDbContext

Entity Configuration
Aggregate Configuration
Child Configuration
Enum Entity Configuration

OrderConfiguration
OrderLineConfiguration
ShippingAddressConfiguration
OrderStatusConfiguration
PaymentTypeConfiguration
CountryConfiguration
```

Tenant-/Audit-Konfiguration nur, wenn die finale Architektur dies benötigt.

## Audit

```text
SaveChangesInterceptor
ICurrentUser
TimeProvider
```

## Application

Mindestens einen Application Service oder Command Handler, der:

1. `OrderRepository` verwendet,
2. Reference Data über den vorgesehenen Cache-Zugriff lädt,
3. ein Order-Aggregate verändert,
4. `SaveChangesAsync()` verwendet.

Zeige denselben Use Case für:

```text
Single-Tenant
```

und:

```text
Multi-Tenant
```

ohne unnötige Tenant-Abhängigkeiten im Single-Tenant-Fall.

---

# 42. PlantUML

Erstelle mehrere separate, **syntaktisch korrekte und direkt kopierbare PlantUML-Diagramme**.

Die Typen sollen nach Abstraktionsgrad angeordnet sein:

```text
oben    = höchste Abstraktion
unten   = konkretere Implementierungen
```

Mindestens folgende Diagramme sind erforderlich:

## 42.1 Entity Architecture

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

Nur tatsächlich verwendete Typen aufnehmen.

Generic Constraints über Notes darstellen.

---

## 42.2 Aggregate

```text
Order
 ├── OrderLine
 └── ShippingAddress
```

Aggregate Boundary klar darstellen.

---

## 42.3 Repository

```text
IOrderRepository
       │
       ▼
     Order
       │
       ├── OrderLine
       └── ShippingAddress
```

Keine Child-Repositories.

---

## 42.4 Enum Entity

```text
OrderStatus enum
       │
       ▼
OrderStatus Entity
       │
       ▼
PostgreSQL
```

inklusive:

```text
Id
Name
Metadata
```

---

## 42.5 Cache

```text
Application
    │
    ▼
ReferenceDataService
    │
    ▼
ReferenceDataCache
   ┌┴┐
 Hit Miss
  │   │
  │   ▼
  │ Repository
  │   │
  │   ▼
  │ EF Core
  │   │
  │   ▼
  │ PostgreSQL
  └───────
```

---

## 42.6 Cache Stampede

Vergleiche:

```text
5 Requests → 5 DB Queries
```

mit:

```text
5 Requests → 1 DB Query → Cache → 5 Results
```

---

## 42.7 Multi-Instance Cache

Zeige:

```text
PostgreSQL
   │
   ├── App A ── Local Cache
   ├── App B ── Local Cache
   └── App C ── Local Cache
```

und:

```text
PostgreSQL
      │
      ▼
 Shared Redis
   /   |   \
 App A App B App C
```

---

## 42.8 EF Configuration

Zeige die tatsächlich verwendete Configuration-Struktur.

Keine künstliche Klassenexplosion erzeugen.

---

## 42.9 Single-Tenant vs. Multi-Tenant

Zeige:

```text
Single-Tenant

Domain
  ↓
Aggregate
  ↓
Repository
  ↓
EF Core / Npgsql
  ↓
PostgreSQL
```

und:

```text
Multi-Tenant

Request
  ↓
Tenant Resolution
  ↓
Tenant Context
  ↓
Application
  ↓
Aggregate
  ↓
Repository / EF Core
  ↓
Tenant Isolation
  ↓
PostgreSQL
```

Die Tenant-Schicht muss klar als Erweiterung der Basisarchitektur erkennbar sein.

---

## 42.10 Gesamtarchitektur

Zeige:

```text
ASP.NET Core
      │
      ▼
Application
 ┌────┴─────────────┐
 │                  │
Commands          Queries
 │                  │
 ▼                  ▼
Aggregate        Query Handler
Repository          │
 │                  ├── EF Core
 ▼                  └── Dapper
Aggregate
 │
 ▼
DbContext
 │
 ▼
Npgsql
 │
 ▼
PostgreSQL


Reference Data
      │
      ▼
ReferenceDataService
      │
      ▼
Cache
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
PostgreSQL
```

Optionale Capabilities sollen sichtbar sein:

```text
Audit
Multi-Tenancy
Caching
```

---

# 43. Repository-Matrix

Erstelle und korrigiere folgende Matrix entsprechend deiner finalen Architektur:

| Typ | Repository | CRUD | Cache | Tenant |
|---|---|---:|---:|---:|
| Aggregate Root | Aggregate Repository | Ja | optional | optional |
| Child Entity | keines | Nein | Nein | über Aggregate |
| Enum Entity | Reference Data Repository | eingeschränkt | Ja | normalerweise Nein |
| Reference Data | Reference Data Repository | fachlich abhängig | optional | optional |
| Value Object | keines | Nein | Nein | über Owner |

Erkläre Abweichungen.

---

# 44. Capability-Matrix

Erstelle am Ende eine Tabelle:

| Capability | Single-Tenant | Multi-Tenant |
|---|---|---|
| Entity | Ja | Ja |
| Aggregate Root | Ja | Ja |
| Child Entity | Ja | Ja |
| Repository | Ja | Ja |
| Audit | optional | optional |
| Tenant Context | Nein | Ja |
| Tenant Entity Interface | Nein | nur wo benötigt |
| Tenant Query Filter | Nein | Ja |
| Tenant Repository | möglichst Nein | nur bei begründetem Bedarf |
| Reference Data Cache | optional | optional |
| Distributed Cache | Deployment-abhängig | Deployment-abhängig |
| PostgreSQL | Ja | Ja |

Korrigiere die Tabelle, falls die finale Architektur zu anderen Ergebnissen kommt.

Die zentrale Aussage muss erhalten bleiben:

> **Multi-Tenancy erweitert die Basisarchitektur, anstatt sie zu definieren.**

---

# 45. Architekturentscheidungen

Bewerte abschließend ausdrücklich:

1. Vererbung vs. Interfaces
2. Composition vs. Vererbung
3. Entity Equality
4. Strongly Typed IDs
5. Aggregate Root
6. Child Entity
7. Aggregate Repository
8. Generic Repository
9. Concrete Repository
10. Tenant Repository
11. Tenant Context
12. Global Query Filters
13. PostgreSQL RLS
14. Audit
15. SaveChangesInterceptor
16. Persistierbare Enum Entity
17. C# Enum vs. Smart Enum
18. Enum Seed Data
19. Enum ID als Primary Key
20. Reference Data Repository
21. Cache Abstraction
22. Cache Decorator
23. Cache-aside
24. Concurrent Cache Misses
25. IMemoryCache
26. Redis
27. Distributed Cache
28. Cache Invalidation
29. Unit of Work
30. DbContext als Unit of Work
31. Specification Pattern
32. CQRS
33. Query Handler
34. Dapper
35. PostgreSQL-spezifische Features
36. Testbarkeit
37. Performance
38. Skalierbarkeit
39. Multi-Instance Deployment
40. Single-Tenant
41. Multi-Tenant

Für jede wesentliche Entscheidung:

```text
Alternativen
    ↓
Vor- und Nachteile
    ↓
Empfehlung
    ↓
Begründung
```

---

# 46. Finale Empfehlung

Die Antwort darf nicht bei einer Sammlung von Klassen stehen bleiben.

Am Ende muss eine **konkrete, zusammenhängende Produktionsarchitektur** empfohlen werden.

Die finale Empfehlung muss eindeutig beantworten:

### Entity

- Wie sieht die finale Entity-Basisklasse aus?
- Welche Interfaces existieren?
- Welche Generic Constraints werden verwendet?
- Wie funktioniert Equality?

### Aggregate

- Wie wird ein Aggregate Root gekennzeichnet?
- Wie werden Child Entities modelliert?
- Wie wird deren Lifecycle gekapselt?
- Warum gibt es keine Child-Repositories?

### Tenant

- Ist Tenant Bestandteil der Basisarchitektur?
- Wie wird Tenant optional aktiviert?
- Gibt es ein Tenant Repository?
- Wo findet Tenant Isolation statt?
- Wird PostgreSQL RLS empfohlen?
- Wie funktioniert dieselbe Architektur ohne Tenant?

### Audit

- Interface oder Vererbung?
- Wie werden Werte automatisch gesetzt?
- Welche Rolle spielt ein EF-Core-Interceptor?

### Enum Entities

- Was ist eine Enum Entity?
- Wie wird der numerische Wert persistiert?
- Wie wird der String gespeichert?
- Wie werden Metadaten gespeichert?
- Wie wird Seed Data synchronisiert?
- Wie werden Inkonsistenzen erkannt?

### Repository

- Was ist tatsächlich ein Repository?
- Was ist Application Service?
- Was ist Reference Data Repository?
- Was ist Cache?
- Was ist Tenant Context?
- Was ist Query Handler?
- Ist eine zusätzliche Unit of Work erforderlich?

### Cache

- Wo befindet sich der Cache?
- Wie funktioniert Cache-aside?
- Wie wird Concurrent Cache Miss verhindert?
- Welche Cache-Technologie wird empfohlen?
- Wie funktioniert die Architektur mit mehreren Instanzen?
- Wie werden Tenant-spezifische Cache Keys behandelt?

### PostgreSQL

- Welche PostgreSQL-Datentypen werden verwendet?
- Wie werden Strongly Typed IDs gespeichert?
- Welche Indizes werden benötigt?
- Welche Unique Constraints werden benötigt?
- Wie werden Foreign Keys definiert?
- Welche PostgreSQL-spezifischen Features werden genutzt?
- Wo ist PostgreSQL-spezifisches Wissen gekapselt?

### Application

- Wie sieht ein Command/Use Case aus?
- Wie werden Aggregate Repository und Reference Data verwendet?
- Wie wird `SaveChangesAsync()` aufgerufen?
- Wie funktioniert der Use Case in Single- und Multi-Tenant?

### Architecture

- Was gehört in Domain?
- Was gehört in Application?
- Was gehört in Infrastructure?
- Wo liegt EF Core?
- Wo liegt Npgsql?
- Wo liegt PostgreSQL?
- Wo liegt Redis?
- Welche Komponenten sind optional?
- Welche Komponenten sind zwingender Bestandteil der Basisarchitektur?

---

# 47. Form der Antwort

Strukturiere die Antwort in logisch zusammenhängende Abschnitte.

Verwende:

- Markdown
- Tabellen
- Textdiagramme
- C#-Code
- PlantUML

PlantUML- und C#-Blöcke müssen direkt kopierbar sein.

Verwende echten C#-Code, soweit dies sinnvoll möglich ist, und vermeide unnötigen Pseudocode.

Architekturentscheidungen sollen **direkt an der relevanten Stelle** erklärt werden.

Verwende für wichtige Entscheidungen:

> **Empfehlung**

und erläutere anschließend kurz:

- warum diese Variante gewählt wurde,
- welche Alternativen existieren,
- welche Nachteile die Alternativen haben.

---

# 48. Wichtigste Gesamtanforderung

Die gesamte Architektur muss diese Beziehung nachvollziehbar machen:

```text
                    ┌──────────────────────┐
                    │  Basisarchitektur    │
                    │                      │
                    │ Entity               │
                    │ Aggregate            │
                    │ Child Entity         │
                    │ Repository           │
                    │ EF Core              │
                    │ Npgsql               │
                    │ PostgreSQL           │
                    └──────────┬───────────┘
                               │
                    optionale Capabilities
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
            Audit        Multi-Tenancy       Cache
```

Insbesondere muss gelten:

```text
Single-Tenant
```

ist ein vollständiger und einfacher Anwendungsfall der Basisarchitektur.

```text
Multi-Tenant
```

ist eine optionale Erweiterung.

Eine Single-Tenant-Anwendung soll nicht gezwungen sein, beispielsweise:

```text
ITenantContext
TenantId
TenantRepository
Tenant Query Filters
Tenant-aware Cache Keys
Tenant Resolution
```

zu verwenden.

Die finale Architektur soll außerdem klar zwischen diesen Verantwortlichkeiten unterscheiden:

```text
Domain
    ↓
Aggregate
    ↓
Application
    ↓
Repository / Query Handler
    ↓
EF Core
    ↓
Npgsql
    ↓
PostgreSQL
```

und:

```text
Reference Data
    ↓
ReferenceDataService
    ↓
Cache
    ↓
ReferenceDataRepository
    ↓
EF Core / Npgsql
    ↓
PostgreSQL
```

sowie optional:

```text
Request
    ↓
Tenant Resolution
    ↓
Tenant Context
    ↓
Tenant Isolation
    ↓
Application / EF Core / PostgreSQL
```

**Ziel ist eine konkrete, konsistente und produktionsgeeignete Architektur – nicht eine möglichst große Sammlung von Abstraktionen.**

Wenn zwei Abstraktionen dieselbe Verantwortung erfüllen, soll ausdrücklich die einfachere Lösung bevorzugt werden.

Wenn eine Abstraktion keinen klaren fachlichen oder technischen Nutzen besitzt, soll sie **nicht** eingeführt werden.
:::

**Warum ich diese Fassung für deutlich besser halte:** Die PostgreSQL-Vorgabe ist jetzt eine echte Architekturvoraussetzung und nicht nur ein nachträglicher Provider-Hinweis. Dadurch kann die Antwort z. B. Strongly Typed IDs, Zeittypen, Indizes, Constraints, Transaktionen und optional RLS konkret für PostgreSQL bewerten. Gleichzeitig bleibt die Domain providerunabhängig.

Außerdem ist die wichtigste Wiederholung aus deinem ursprünglichen Prompt beseitigt: **Single-Tenant vs. Multi-Tenant wird jetzt einmal als übergreifendes Architekturprinzip definiert und anschließend nur dort konkretisiert, wo es technisch relevant ist.** Das Gleiche gilt für Repository, Cache, Audit und Enum Entities.

Ein weiterer Vorteil: Der neue Prompt fordert nicht mehr implizit, dass alle von dir genannten Abstraktionen auch tatsächlich implementiert werden müssen. Das Modell darf beispielsweise zu dem Ergebnis kommen, dass **kein `ITenantRepository`, kein `ICacheRepository` und keine zusätzliche `IUnitOfWork`** sinnvoll sind. Das ist für die gewünschte Architekturentscheidung wichtig.

Für PostgreSQL würde ich ausdrücklich beim **Npgsql-EF-Core-Provider** bleiben; Microsoft führt `Npgsql.EntityFrameworkCore.PostgreSQL` als PostgreSQL-Provider und weist darauf hin, dass EF-Core-Provider an ihre jeweilige EF-Core-Hauptversion gekoppelt sind. 

Wenn du möchtest, kann ich im nächsten Schritt aus dieser bereinigten Fassung noch eine **„Version 2 – maximal präziser Architektur-Prompt“** machen, die zusätzlich verhindert, dass das Modell bei der eigentlichen Antwort in typische DDD-/Generic-Repository-/EF-Core-Widersprüche läuft.