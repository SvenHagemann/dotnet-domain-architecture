# Aufgabe

Ich möchte eine C#-Webapplikation mit ASP.NET Core und Entity Framework Core, mittels einer sauberen, generischen und langfristig wartbaren Domain- und Persistence-Architektur erarbeiten und am Ende das Ganze über ein oder mehrere PlantUML-Diagramme visualisieren.
Geordnet sollen die Typen im PlantUML-Diagramm nach Abstraktions-Level - oben die höchste Abstraktion und nach Unten hin die immer konkreteren Typen.
Es handelt sich um eine verteilte E-Commerce Applikation, u.a. mit den Services:
- Basket
- Catalog
- Discount
- Ordering
- Payment
- Auth

Bitte entwirf eine **robuste, praxisnahe und konsistente Gesamtarchitektur** für:

- generische Entities
- Strongly Typed IDs
- Aggregate Roots
- Child Entities
- persistierbare typisierte Enum Entities
- Stammdaten
- optionale Multi-Tenancy
- optionales Auditing
- Repositories
- Reference-Data-Repositories
- Caching
- EF-Core-Konfigurationen
- Unit of Work
- optional CQRS / Query Handler
- Dependency Injection

Die Architektur soll nicht nur aus einzelnen Klassen bestehen. Entwickle eine **zusammenhängende Architektur**, erkläre die wichtigsten Designentscheidungen und stelle verschiedene sinnvolle Alternativen gegenüber.

Treffe anschließend eine **klare Empfehlung für eine produktionsgeeignete Lösung**.

---

# 1. Wichtigste architektonische Anforderung: Single-Tenant und Multi-Tenant

Die Architektur muss ausdrücklich für **beide Szenarien** geeignet sein:

1. Anwendungen ohne Multi-Tenancy
2. Anwendungen mit Multi-Tenancy

Dabei soll **Single-Tenancy der einfache Basisfall** sein.

Multi-Tenancy soll als **optionale Capability / Cross-Cutting Concern** ergänzt werden können, ohne die gesamte Entity-, Repository- und Domain-Struktur neu entwerfen zu müssen.

Das bedeutet insbesondere:

```text
Single-Tenant
    │
    ├── Entity
    ├── Aggregate Root
    ├── Child Entity
    ├── Audit
    ├── Repository
    └── EF Core
```

und optional:

```text
Multi-Tenant
    │
    ├── Entity
    ├── Aggregate Root
    ├── Child Entity
    ├── Tenant Awareness
    ├── Audit
    ├── Repository
    └── EF Core + Tenant Isolation
```

Die Tenant-Funktionalität darf **nicht zur Voraussetzung der gesamten Architektur** werden.

Eine Single-Tenant-Anwendung soll insbesondere nicht gezwungen sein, folgende Komponenten zu verwenden:

```text
ITenantContext
ITenantEntity<TTenantKey>
TenantId
TenantRepository
Tenant Query Filters
Tenant-aware Cache Keys
Tenant Resolution
```

wenn keine Multi-Tenancy benötigt wird.

---

# 2. Optionalität von Multi-Tenancy

Untersuche verschiedene Möglichkeiten, Tenant-Unterstützung optional zu gestalten.

Beispielsweise:

```csharp
public interface ITenantEntity<TTenantKey>
    where TTenantKey : struct
{
    TTenantKey TenantId { get; }
}
```

oder:

```csharp
public interface ITenantAware
{
}
```

oder eine andere geeignete Lösung.

Bewerte insbesondere:

- Interface
- Basisklasse
- Composition
- Generic Constraint
- EF-Core-Konfiguration
- Repository Decorator
- Global Query Filter
- Tenant Context
- Application Service

Vermeide eine Architektur, in der beispielsweise jede Entity zwingend von:

```csharp
TenantEntity<TKey, TTenantKey>
```

ableiten muss.

Eine Entity soll auch problemlos so aussehen können:

```csharp
public sealed class Product
    : AggregateRoot<ProductId>
{
}
```

ohne irgendeine Tenant-Abhängigkeit.

Eine Tenant-fähige Entity könnte dagegen beispielsweise sein:

```csharp
public sealed class Order
    : AggregateRoot<OrderId>,
      ITenantEntity<TenantId>
{
}
```

Die konkrete Lösung soll von dir beurteilt und gegebenenfalls anders umgesetzt werden.

---

# 3. Tenant als Capability

Behandle Multi-Tenancy als **optionale Capability**, ähnlich wie Audit.

Die Architektur sollte konzeptionell ermöglichen:

```text
                 Entity
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
     Aggregate   Auditable   Tenant
       Root
        │
        ▼
      Child
```

Dabei können verschiedene Kombinationen entstehen:

```text
Aggregate Root
Aggregate Root + Audit
Aggregate Root + Tenant
Aggregate Root + Tenant + Audit

Child Entity
Child Entity + Audit
Child Entity + Tenant
Child Entity + Tenant + Audit
```

Vermeide hierfür eine Klassenexplosion wie:

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

wenn Interfaces, Composition oder generische Basisklassen dies sauberer lösen.

---

# 4. Grundlegendes Ziel

Die Entities sollen möglichst generisch aufgebaut sein und über Vererbung, Interfaces und Generic Constraints weiter spezialisiert werden können.

Eine mögliche Zielstruktur:

```text
Entity<TKey>
    │
    ├── AggregateRoot<TKey>
    │
    └── ChildEntity<TKey>
```

mit optionalen Capabilities:

```text
IAuditableEntity<TUserKey>
ITenantEntity<TTenantKey>
```

und Reference Data:

```text
IReferenceDataEntity<TKey>
EnumEntity<...>
```

Diese Struktur ist lediglich ein Ausgangspunkt.

Entscheide selbst, welche Elemente tatsächlich als:

- Basisklasse
- Interface
- Value Object
- Composition
- Infrastructure Service

modelliert werden sollten.

---

# 5. Ausgangspunkt: Entity

Als Ausgangspunkt soll folgende Entity-Struktur betrachtet und kritisch bewertet werden:

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

        // Transient objects are not considered equal.
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

Beurteile diese Implementierung kritisch.

Untersuche insbesondere:

- Entity Equality
- HashCode
- Vererbung
- polymorphe Entities
- EF-Core-Proxies
- transient entities
- `init`-Properties
- Strongly Typed IDs
- Persistence Concerns
- Domain Concerns
- Verwendung von Entities in Collections
- Verwendung von Entities als Dictionary Keys
- Auswirkungen eines wechselnden Proxy-Typs auf Equality
- Auswirkungen der Entity-ID auf Equality und HashCode

Wenn Änderungen notwendig sind, liefere eine verbesserte Version und erkläre genau, welches Problem dadurch gelöst wird.

---

# 6. Strongly Typed IDs

Berücksichtige Strongly Typed IDs.

Beispielsweise:

```csharp
public readonly record struct OrderId(Guid Value);

public readonly record struct OrderLineId(Guid Value);

public readonly record struct TenantId(Guid Value);

public readonly record struct UserId(Guid Value);

public readonly record struct OrderStatusId(int Value);
```

Zeige, wie diese mit:

```csharp
Entity<TKey>
```

zusammenarbeiten.

Untersuche:

- `where TKey : struct`
- zusätzliche Generic Constraints
- Value Objects
- EF-Core Value Converters
- Primary Keys
- Foreign Keys
- Composite Keys
- Enum Keys
- Performance
- Typsicherheit

Zeige insbesondere, wie verhindert wird, dass versehentlich:

```csharp
OrderId
```

an eine Methode übergeben wird, die:

```csharp
CustomerId
```

erwartet.

---

# 7. Aggregate Roots

Entwickle eine generische Struktur für Aggregate Roots.

Beispielsweise:

```csharp
public interface IAggregateRoot<TKey> : IEntity<TKey>
    where TKey : struct
{
}
```

und gegebenenfalls:

```csharp
public abstract class AggregateRoot<TKey> :
    Entity<TKey>,
    IAggregateRoot<TKey>
    where TKey : struct
{
}
```

Dies ist nur ein Ausgangspunkt.

Entscheide selbst, ob diese Struktur sinnvoll ist.

Erkläre:

- Was ein Aggregate Root von einer normalen Entity unterscheidet
- Warum Aggregate Roots die Repository-Grenze darstellen
- Warum nicht jede Entity ein Repository benötigt
- Wie Aggregate Boundaries im Type-System dargestellt werden können
- Welche Dinge C#-Generic-Constraints verhindern können
- Welche Dinge sich nicht vollständig durch das Type-System verhindern lassen

---

# 8. Child Entities

Ein Aggregate soll aus einem Aggregate Root und beliebig vielen untergeordneten Entities bestehen können.

Beispiel:

```text
Order
 ├── OrderLine
 ├── OrderLine
 └── ShippingAddress
```

`Order` ist Aggregate Root.

`OrderLine` und `ShippingAddress` gehören ausschließlich zu diesem Aggregate.

Sie sollen **keine eigenen Repositories** besitzen.

Gewünschtes Modell:

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

Nicht:

```text
IOrderRepository
IOrderLineRepository
IShippingAddressRepository
```

Entwickle dafür passende Interfaces und Generic Constraints.

Beispielsweise:

```csharp
public interface IChildEntity<TKey> : IEntity<TKey>
    where TKey : struct
{
}
```

und gegebenenfalls:

```csharp
public abstract class ChildEntity<TKey> :
    Entity<TKey>,
    IChildEntity<TKey>
    where TKey : struct
{
}
```

Bewerte auch, ob `ChildEntity<TKey>` eine sinnvolle Abstraktion ist oder ob ein Marker-Interface genügt.

---

# 9. Aggregate API

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

durch beliebigen Application-Code.

Zeige, wie Collections gekapselt werden können:

```csharp
private readonly List<OrderLine> _lines = [];

public IReadOnlyCollection<OrderLine> Lines =>
    _lines;
```

Erkläre:

- Invarianten
- Aggregate Encapsulation
- Child Lifecycle
- Domain Methods
- Navigation Properties
- EF Core Compatibility

---

# 10. Optionales Tenant-Konzept

Entwickle ein Tenant-Konzept, das **nicht Teil der Basisklasse aller Entities** sein muss.

Beispielsweise:

```csharp
public interface ITenantEntity<TTenantKey>
    where TTenantKey : struct
{
    TTenantKey TenantId { get; }
}
```

Entwickle daraus eine sinnvolle generische Struktur.

Berücksichtige:

- TenantId
- Generic Constraints
- Aggregate Roots
- Child Entities
- Strongly Typed Tenant IDs
- EF-Core-Konfiguration
- Global Query Filters
- Unique Constraints
- Indexes
- Composite Keys
- Tenant Isolation
- Cross-Tenant References

Untersuche insbesondere, ob `TenantId` Bestandteil des Primary Keys sein sollte.

Vergleiche:

```text
PK = EntityId
TenantId = separate column
```

mit:

```text
PK = TenantId + EntityId
```

und entscheide dich begründet.

Wichtig:

Für eine Single-Tenant-Anwendung soll diese gesamte Tenant-Schicht **nicht erforderlich** sein.

---

# 11. Tenant Isolation

Nur wenn Multi-Tenancy aktiviert ist, muss die Architektur verhindern, dass ein Tenant versehentlich auf Daten eines anderen Tenants zugreift.

Beispielsweise:

```text
Tenant A
 └── Order A
      └── OrderLine A

Tenant B
 └── Order B
      └── OrderLine B
```

Ein Zugriff aus Tenant A darf niemals:

```text
Order B
```

zurückgeben.

Untersuche dafür:

- `ITenantContext`
- Application Layer
- Repository
- EF-Core Global Query Filters
- EF-Core Interceptors
- Database Constraints
- Row-Level Security
- Repository Decorators

Erkläre, welche Ebene für welche Verantwortung zuständig sein sollte.

Unterscheide ausdrücklich zwischen:

```text
Tenant Resolution
Tenant Filtering
Tenant Authorization
Tenant Data Isolation
```

Tenant-Isolation soll als **Security Requirement** behandelt werden, nicht lediglich als Komfortfunktion.

Für Single-Tenant-Anwendungen sollen diese Mechanismen vollständig deaktivierbar bzw. gar nicht erst erforderlich sein.

---

# 12. Audit

Bestimmte Entities sollen Audit-Informationen besitzen.

Beispielsweise:

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

Entwickle hierfür eine sinnvolle Struktur.

Berücksichtige:

- CreatedAt
- CreatedBy
- ModifiedAt
- ModifiedBy
- optional Tenant
- Aggregate Root
- Child Entity
- EF Core
- SaveChanges
- SaveChangesAsync
- SaveChanges Interceptor
- Current User Context
- Time Provider
- Domain vs. Infrastructure

Bewerte insbesondere, ob Audit-Funktionalität Bestandteil der Vererbungshierarchie sein sollte oder besser über Interfaces und Infrastructure umgesetzt wird.

Vermeide eine Klassenexplosion wie:

```text
AuditableEntity
TenantEntity
AuditableTenantEntity
AuditableTenantAggregateRoot
TenantAggregateRoot
...
```

wenn dies über Interfaces/Composition sauberer lösbar ist.

Audit soll ebenfalls optional sein.

---

# 13. Persistierbare Enum Entities

Der Begriff "Enum Entity" bedeutet ausdrücklich **keine normale Lookup Entity**.

Gemeint ist ein persistierbares, typisiertes Enum-Konzept.

Ein Enum-Wert besitzt:

```text
numerischer Wert
      ↓
Primary Key / Identität

String-Wert
      ↓
persistierter Wert / Name

weitere Spalten
      ↓
fachliche Metadaten
```

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

Daraus soll eine persistierte Struktur entstehen können:

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
- `Id` ist der Primary Key.
- `Id` ist stabil.
- EF Core darf `Id` nicht generieren.
- Der String-Wert soll persistiert werden.
- Weitere fachliche Spalten sind erlaubt.
- Die Werte werden durch Code/Enum definiert.
- Benutzer sollen keine neuen Werte über normales CRUD anlegen.
- Die Datenbank repräsentiert diese Werte relational.
- Andere Entities dürfen über den numerischen Wert referenzieren.
- Die Enum Entity benötigt kein normales CRUD-Repository.

---

# 14. Enum Entity vs. Lookup Entity

Erkläre ausdrücklich den Unterschied zwischen:

```text
Lookup Entity
```

und:

```text
persistierbare Enum Entity
```

Eine Lookup Entity könnte beispielsweise von einem Benutzer angelegt werden:

```text
Country
PaymentMethod
CustomerCategory
```

Eine Enum Entity hingegen wird durch Code definiert:

```text
OrderStatus
    Pending
    Confirmed
    Shipped
    Cancelled
```

Weitere Spalten dürfen zusätzliche fachliche Eigenschaften des Enum-Wertes enthalten.

Beispielsweise:

```text
OrderStatus
 ├── Id
 ├── Name
 ├── IsFinal
 ├── AllowsModification
 ├── SortOrder
 └── ...
```

Erkläre, warum diese Entity nicht wie eine normale CRUD-Entity behandelt werden sollte.

---

# 15. Ansätze für Enum Entities

Bewerte mindestens diese Ansätze:

## Ansatz A – C# Enum + separate Entity

```csharp
public enum OrderStatus
{
    Pending = 1,
    Confirmed = 2,
    Shipped = 3
}
```

plus EF-Core Entity.

## Ansatz B – Smart Enum / Enumeration Pattern

```csharp
public sealed class OrderStatus
{
    public static readonly OrderStatus Pending =
        new(1, "Pending");

    public static readonly OrderStatus Confirmed =
        new(2, "Confirmed");

    public int Id { get; }

    public string Name { get; }
}
```

## Ansatz C – Generische EnumEntity

```csharp
public abstract class EnumEntity<TKey, TEnum>
    : Entity<TKey>
    where TKey : struct
    where TEnum : struct, Enum
{
    public string Name { get; protected init; } = null!;
}
```

Wenn keiner dieser Ansätze optimal ist, entwickle eine bessere Alternative.

Berücksichtige:

- Typsicherheit
- numerische Identität
- String-Repräsentation
- zusätzliche Metadaten
- EF Core
- Seed Data
- Migrationen
- `ValueGeneratedNever()`
- Stabilität der IDs
- Synchronisierung zwischen Enum und Datenbank
- Repository
- Cache
- Performance
- Testbarkeit

Diskutiere außerdem:

```text
Enum.ToString()
```

gegenüber einem explizit persistierten String-Wert.

---

# 16. Synchronisierung Enum ↔ Datenbank

Die Architektur soll erklären, wie sichergestellt wird, dass:

```text
C# Enum
    ↕
Database Seed Data
```

konsistent bleiben.

Untersuche:

- EF Core `HasData`
- JSON Seed Data
- Migration Seed
- Custom Seed Extension
- Startup Validation
- Integration Tests
- Build-Time Validation
- Runtime Validation

Beispielsweise:

```text
C# Enum
 ├── Pending = 1
 ├── Confirmed = 2
 └── Shipped = 3

Database
 ├── 1 Pending
 ├── 2 Confirmed
 └── 3 Shipped
```

Was soll passieren, wenn:

```text
C#:
Shipped = 3

Database:
Shipped = 4
```

oder:

```text
C#:
Cancelled = 4

Database:
Cancelled fehlt
```

Definiere eine robuste Strategie.

---

# 17. Repository Architecture

Entwickle zusätzlich eine **konsistente generische Repository-Architektur**.

Wichtig:

Nimm nicht automatisch an, dass jede technische Abstraktion ein Repository sein muss.

Untersuche kritisch folgende möglichen Konzepte:

```text
Repository
Aggregate Repository
ReferenceDataRepository
TenantRepository
CacheRepository
Cache
TenantContext
UnitOfWork
```

Die Antwort soll ausdrücklich klären:

> Welche dieser Abstraktionen sind tatsächlich Repositorys und welche sollten besser als Service, Context, Decorator oder Infrastructure Component modelliert werden?

---

# 18. Generic Aggregate Repository

Entwickle ein generisches Repository für Aggregate Roots.

Beispielsweise:

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

Dies ist nur ein Ausgangspunkt.

Entwickle eine bessere Variante, falls erforderlich.

Das Repository soll:

- nur Aggregate Roots akzeptieren
- Child Entities ausschließen
- Aggregate Loading unterstützen
- Tracking kontrollieren
- `AsNoTracking` sinnvoll einsetzen
- Cancellation Tokens unterstützen
- async/await verwenden
- EF Core unterstützen
- Unit of Work berücksichtigen
- keine unnötigen Transaktionen verstecken
- keine Infrastrukturdetails an die Domain geben

---

# 19. Generic Repository vs. Concrete Repository

Vergleiche:

### Variante A

```text
IRepository<TAggregate, TKey>
        │
        └── IOrderRepository
```

### Variante B

```text
IOrderRepository
    └── eigene Implementierung
```

### Variante C

```text
Generic Repository
        │
        ├── Standardoperationen
        │
        └── Concrete Repository
                └── spezielle Queries
```

Bewerte:

- Typsicherheit
- Wartbarkeit
- Testbarkeit
- Flexibilität
- DDD
- EF Core
- Query-Komplexität
- Gefahr eines Generic Repository Anti-Patterns

Gib eine klare Empfehlung.

---

# 20. Concrete Aggregate Repositories

Zeige mindestens:

```csharp
public interface IOrderRepository
    : IRepository<Order, OrderId>
{
}
```

und eine konkrete Implementierung.

Falls sinnvoll:

```csharp
public sealed class OrderRepository :
    Repository<Order, OrderId>,
    IOrderRepository
{
}
```

Falls dies nicht empfohlen wird, zeige die bessere Variante.

Das Repository soll auf Aggregate-Ebene arbeiten.

---

# 21. Child Entity Repository

Es darf kein eigenes Repository für:

```text
OrderLine
ShippingAddress
```

geben.

Insbesondere soll folgendes nicht Teil der Architektur sein:

```csharp
IRepository<OrderLine, OrderLineId>
```

Erkläre, warum die Generic Constraints des Aggregate Repositorys dies bereits weitgehend verhindern können.

Erkläre aber auch die Grenzen des C#-Typsystems.

---

# 22. Tenant Repository – kritisch bewerten

Untersuche kritisch, ob ein:

```csharp
ITenantRepository<TAggregate, TKey, TTenantKey>
```

sinnvoll ist.

Beispielsweise:

```csharp
public interface ITenantRepository<
    TAggregate,
    TKey,
    TTenantKey>
    : IRepository<TAggregate, TKey>
    where TAggregate :
        class,
        IAggregateRoot<TKey>,
        ITenantEntity<TTenantKey>
    where TKey : struct
    where TTenantKey : struct
{
    Task<TAggregate?> GetByIdAsync(
        TTenantKey tenantId,
        TKey id,
        CancellationToken cancellationToken = default);
}
```

Dies ist ausdrücklich nur ein Beispiel.

Bewerte, ob Tenant-Semantik besser über:

- `ITenantContext`
- Global Query Filters
- Repository Decorator
- Specification
- Application Service
- EF Core Interceptor
- Database Row-Level Security

gelöst werden sollte.

Es ist ausdrücklich erlaubt und erwünscht, zu dem Ergebnis zu kommen:

```text
Kein separates TenantRepository erforderlich.
```

Falls dies die bessere Architektur ist, erkläre warum.

Wichtig:

In einer Single-Tenant-Anwendung soll dieses Konzept **nicht erforderlich** sein.

---

# 23. Reference Data Repository

Für persistierbare Enum Entities und Stammdaten soll ein eigenes Konzept untersucht werden.

Beispielsweise:

```csharp
IReferenceDataRepository<T, TKey>
```

mit:

```csharp
where T : class, IReferenceDataEntity<TKey>
where TKey : struct
```

Mögliche Methoden:

```csharp
Task<T?> GetByIdAsync(...);

Task<IReadOnlyList<T>> GetAllAsync(...);
```

Das Repository soll **nicht als normales CRUD-Repository** betrachtet werden.

Insbesondere soll nicht automatisch enthalten sein:

```text
AddAsync()
UpdateAsync()
DeleteAsync()
```

wenn diese Operationen für die betreffende Stammdatenart fachlich nicht vorgesehen sind.

---

# 24. Cache Repository vs. Cache Abstraction

Untersuche ausdrücklich, ob:

```csharp
ICacheRepository<T, TKey>
```

überhaupt sinnvoll ist.

Vergleiche:

## Variante A

```text
ICacheRepository<T>
```

## Variante B

```text
IReferenceDataCache<T, TKey>
```

## Variante C

```text
CachedReferenceDataRepository<T>
```

als Decorator.

## Variante D

```text
ReferenceDataService
        │
        ├── Cache
        └── Repository
```

Bewerte die Varianten.

Die Architektur soll möglichst klar zwischen:

```text
Persistence
    ≠
Caching
```

unterscheiden.

Ein Cache soll nicht als Datenbankersatz betrachtet werden.

---

# 25. Bevorzugte Cache-Architektur

Wenn ein Decorator sinnvoll ist, untersuche beispielsweise:

```text
                 Application
                     │
                     ▼
        IReferenceDataRepository<T>
                     ▲
                     │
              ┌──────┴──────┐
              │             │
        Cache Decorator   EF Repository
              │             │
              ▼             ▼
            Cache         EF Core
                            │
                            ▼
                         Database
```

Alternativ:

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
Database
```

Entscheide dich für einen Ansatz und begründe ihn.

---

# 26. Stammdaten-Caching

Stammdaten sollen beim ersten benötigten Zugriff aus der Datenbank geladen und anschließend gecacht werden.

Gewünschtes Verhalten:

```text
Erste Anfrage

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

Weitere Anfragen:

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

Es soll also gelten:

```text
First Access:
Database → Cache

All Further Access:
Cache → Application
```

Weitere Requests sollen **keine Datenbankabfrage** auslösen, solange der Cache-Eintrag gültig ist.

---

# 27. Cache API

Entwickle eine generische Cache-Abstraktion.

Beispielsweise:

```csharp
public interface IReferenceDataCache<T, TKey>
    where T : class
    where TKey : struct
{
    Task<T?> GetAsync(
        TKey id,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyCollection<T>> GetAllAsync(
        CancellationToken cancellationToken = default);
}
```

Dies ist nur ein Beispiel.

Entwickle die endgültige API selbst.

Sie soll beispielsweise ermöglichen:

```text
IReferenceDataCache<OrderStatus, OrderStatusId>
IReferenceDataCache<PaymentType, PaymentTypeId>
IReferenceDataCache<Country, CountryId>
```

---

# 28. Cache Stampede / Concurrent Requests

Berücksichtige unbedingt parallele Requests.

Vermeide:

```text
Request A ─┐
Request B ─┤
Request C ─┼── Cache Miss ──> 5x Database Query
Request D ─┤
Request E ─┘
```

Stattdessen:

```text
Request A ─┐
Request B ─┤
Request C ─┼── Cache Miss ──> 1x Database Query
Request D ─┤                         │
Request E ─┘                         ▼
                                  Cache
                                     │
                      ┌──────────────┼──────────────┐
                      ▼              ▼              ▼
                   Request A      Request B      Request C
```

Bewerte und implementiere einen geeigneten Mechanismus.

Untersuche:

- `Lazy<Task<T>>`
- `SemaphoreSlim`
- `ConcurrentDictionary`
- Locks
- Single Flight
- Async Locks

Vermeide:

- Deadlocks
- Race Conditions
- unnötige globale Locks
- unnötige Synchronisationsengpässe

---

# 29. Cache-Technologie

Vergleiche:

- `IMemoryCache`
- `IDistributedCache`
- Redis
- Application-Level Cache
- Cache-aside
- Read-through
- Write-through
- Repository Decorator
- Startup Preloading
- Lazy Loading
- EF-Core Second-Level Cache

Entscheide dich für eine bevorzugte Lösung.

Berücksichtige:

- Anzahl Application-Instanzen
- Containerisierung
- Skalierung
- Cache Consistency
- Speicherverbrauch
- Latenz
- Betriebskomplexität
- Ausfallsicherheit

Zeige ausdrücklich, wie sich die Entscheidung zwischen Single-Tenant und Multi-Tenant auf den Cache auswirkt.

---

# 30. Cache Key und optionaler Tenant

Definiere eine konsistente Cache-Key-Strategie.

Für Single-Tenant beispielsweise:

```text
reference-data:OrderStatus
reference-data:PaymentType
reference-data:Country
```

Für Multi-Tenant könnte gegebenenfalls erforderlich sein:

```text
tenant:{tenantId}:reference-data:...
```

Entscheide, ob Tenant Bestandteil des Cache Keys sein muss.

Wichtig:

Eine Single-Tenant-Anwendung soll keine künstliche Tenant-Komponente in ihren Cache Keys benötigen.

Berücksichtige:

- Namespace
- Entity Type
- Key
- optional Tenant
- Version
- Environment
- Cache Invalidation

---

# 31. Lazy Loading vs. Startup Preloading

Vergleiche:

## Lazy Loading

```text
Application
    │
    ▼
Get<OrderStatus>()
    │
    ▼
Cache Miss
    │
    ▼
Database
```

## Startup Loading

```text
Application Startup
       │
       ▼
ReferenceDataLoader
       │
       ├── OrderStatus
       ├── PaymentType
       ├── Country
       └── Currency
              │
              ▼
            Cache
```

## Hybrid

```text
Application Startup
       │
       ▼
Preload wichtiger Stammdaten
       │
       ▼
     Cache
       │
       └── Lazy Loading für seltene Daten
```

Vergleiche die Ansätze und gib eine Empfehlung.

---

# 32. Cache Invalidierung

Berücksichtige:

- TTL
- Explicit Invalidation
- Cache Refresh
- Application Restart
- Deployment
- Migration
- Seed Data
- Event-based Invalidation
- Distributed Invalidation

Auch wenn Stammdaten normalerweise selten geändert werden, muss beschrieben werden, was bei einer Änderung passiert.

---

# 33. Mehrere Application-Instanzen

Berücksichtige:

```text
                 Database
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
     App A        App B        App C
       │            │            │
    Cache A      Cache B      Cache C
```

und:

```text
                 Database
                    │
                    ▼
               Shared Cache
                /    |    \
               ▼     ▼     ▼
             App A App B App C
```

Vergleiche:

- Local Memory Cache
- Distributed Cache
- Redis
- Cache Consistency
- Invalidierung
- Netzwerk-Latenz
- Ausfallszenarien

Entscheide, welche Lösung für eine skalierbare Webapplikation sinnvoll ist.

---

# 34. EF Core und Cache

Die Domain soll nichts über den Cache wissen.

EF Core soll möglichst nichts über den Application Cache wissen.

Bevorzugte Trennung:

```text
Domain
   │
   ▼
Application
   │
   ▼
Infrastructure
   ├── EF Core
   ├── Database
   └── Cache
```

Untersuche insbesondere:

```text
ReferenceDataService
        │
        ▼
ReferenceDataCache
        │
        │ Miss
        ▼
ReferenceDataRepository
        │
        ▼
EF Core
        │
        ▼
Database
```

und entscheide, ob diese oder eine andere Architektur vorzuziehen ist.

---

# 35. Unit of Work

Untersuche:

```text
Repository
DbContext
Unit of Work
Transaction
```

EF Core `DbContext` übernimmt bereits Repository-/Unit-of-Work-artige Aufgaben.

Führe deshalb nicht automatisch eine zusätzliche `IUnitOfWork` ein.

Bewerte:

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
                │
                ▼
             DbContext
```

Entscheide dich für eine Variante.

---

# 36. Specification Pattern

Untersuche, ob ein Specification Pattern sinnvoll ist.

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
- optional Tenant Filter
- Tracking
- `AsNoTracking`
- Projektionen

Vermeide, dass ein Repository einfach:

```csharp
IQueryable<T>
```

nach außen gibt und dadurch die Persistence-Abstraktion praktisch aufgehoben wird.

Untersuche alternativ:

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

---

# 37. CQRS

Berücksichtige CQRS.

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

Bewerte, ob alle Reads über Repositorys laufen müssen.

Eine mögliche Architektur:

```text
                  Application
                 /           \
                /             \
           Commands           Queries
              │                  │
              ▼                  ▼
        Repositories        Query Handlers
              │                  │
              ▼                  ├── EF Core
         Aggregates              └── Dapper
              │
              ▼
          DbContext
```

Gib eine klare Empfehlung.

---

# 38. Beispiel-Domain

Verwende für die konkrete Demonstration eine kleine Order-Domain:

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

Dabei gilt:

- `Order` ist Aggregate Root.
- `OrderLine` ist Child Entity.
- `ShippingAddress` ist Child Entity oder Value Object; entscheide und begründe.
- `Order` ist optional Tenant-aware.
- `Order` ist optional Auditable.
- `OrderLine` ist optional Tenant-aware und Auditable, sofern dies fachlich sinnvoll ist.
- `OrderLine` besitzt kein eigenes Repository.
- `ShippingAddress` besitzt kein eigenes Repository.
- `OrderRepository` arbeitet auf Aggregate-Ebene.
- `Order.StatusId` verweist auf eine persistierbare Enum Entity.
- `OrderStatus` wird nicht über normales CRUD verwaltet.
- `OrderStatus` wird durch C# Enum/Code und Seed Data definiert.
- `OrderStatus.Id` entspricht dem numerischen Enum-Wert.
- `OrderStatus.Name` enthält dessen String-Repräsentation.
- `OrderStatus` besitzt zusätzliche fachliche Metadaten.
- `OrderStatus` wird aus dem Cache gelesen.
- `PaymentType` und `Country` sollen ebenfalls als Reference Data betrachtet werden.

Zeige die Domain sowohl für:

```text
Single-Tenant
```

als auch für:

```text
Multi-Tenant
```

und stelle heraus, welche Unterschiede tatsächlich notwendig sind.

---

# 39. EF-Core-Beziehungen

Zeige:

```text
Order
  │
  ├── 1:N OrderLine
  │
  ├── 1:0..1 ShippingAddress
  │
  └── N:1 OrderStatus
```

und erkläre:

- Foreign Keys
- Navigation Properties
- required / optional
- Cascade Delete
- DeleteBehavior
- Value Converters
- Owned Entities
- Owned Collections
- Aggregate Boundaries

Entscheide insbesondere, ob `ShippingAddress` besser als:

```text
Child Entity
```

oder:

```text
Value Object / Owned Entity
```

modelliert werden sollte.

---

# 40. EF-Core Configuration

Für jede Entity-Art soll es eine korrespondierende generische Configuration-Struktur geben.

Ausgangspunkt:

```csharp
public abstract class EntityConfiguration<T, TKey>
    : IEntityTypeConfiguration<T>
    where T : class, IEntity<TKey>
    where TKey : struct
{
    protected virtual bool HasGeneratedId => true;

    protected virtual string EntityName =>
        typeof(T).Name;

    public virtual void Configure(
        EntityTypeBuilder<T> builder)
    {
        if (HasGeneratedId)
        {
            builder
                .Property(e => e.Id)
                .ValueGeneratedOnAdd()
                .HasColumnOrder(0);
        }
        else
        {
            builder
                .Property(e => e.Id)
                .ValueGeneratedNever()
                .HasColumnOrder(0);
        }

        builder.HasKey(e => e.Id);

        builder.SeedFromJsonFile();
    }
}
```

Entwickle daraus eine sinnvolle Configuration-Architektur.

Untersuche beispielsweise:

```text
EntityConfiguration<T,TKey>
        │
        ├── AggregateRootConfiguration<T,TKey>
        ├── ChildEntityConfiguration<T,TKey>
        ├── EnumEntityConfiguration<...>
        ├── Tenant Configuration
        └── Audit Configuration
```

Wichtig:

Die Tenant Configuration soll nur dann benötigt werden, wenn die konkrete Entity tatsächlich Tenant-aware ist.

Vermeide:

```text
TenantAuditableAggregateRootConfiguration
TenantAuditableChildEntityConfiguration
...
```

wenn Interfaces, Helper Methods oder Configuration Components geeigneter sind.

---

# 41. Configuration Composition

Zeige, wie mehrere Aspekte konfiguriert werden können.

Beispielsweise:

```text
OrderConfiguration
    │
    ├── Base Entity Configuration
    ├── Aggregate Configuration
    ├── optional Tenant Configuration
    ├── optional Audit Configuration
    └── Order-specific Configuration
```

Entscheide, ob dies über:

- Vererbung
- Helper Methods
- Configuration Components
- Interfaces
- Extension Methods

gelöst werden sollte.

---

# 42. Repository Matrix

Erstelle eine Übersicht:

| Entity-Typ | Repository | CRUD | Cache | Tenant |
|---|---|---:|---:|---:|
| Aggregate Root | Aggregate Repository | Ja | optional | optional |
| Child Entity | keines | Nein | Nein | über Aggregate |
| Enum Entity | Reference Data Repository | Nein | Ja | normalerweise Nein |
| Reference Data | Reference Data Repository | eingeschränkt | Ja | optional |
| Value Object | keines | Nein | Nein | über Owner |

Prüfe diese Matrix kritisch und korrigiere sie gegebenenfalls.

---

# 43. Repository Architecture Diagram

Erstelle ein eigenes PlantUML-Diagramm.

Es soll ungefähr darstellen:

```text
                         Application
                              │
               ┌──────────────┴──────────────┐
               │                             │
           Commands                       Queries
               │                             │
               ▼                             ▼
        Aggregate Repository            Query Handler
               │                             │
               ▼                             ├── EF Core
          Aggregate                         └── Dapper
               │
               ▼
           DbContext
               │
               ▼
           Database


Reference Data:

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

Zeige darin die tatsächlich empfohlenen Interfaces:

```text
IRepository<TAggregate,TKey>
IReferenceDataRepository<T,TKey>
IReferenceDataCache<T,TKey>
IOrderRepository
ITenantRepository<...>       // nur falls empfohlen
IUnitOfWork                  // nur falls empfohlen
ITenantContext               // nur bei Multi-Tenancy
```

---

# 44. Textdiagramme

Verwende zusätzlich zu PlantUML einfache Textdiagramme, wenn diese Sachverhalte besser erklären.

Beispiel Aggregate:

```text
                    Aggregate Root
                          │
              ┌───────────┴───────────┐
              │                       │
          Child Entity           Child Entity
              │                       │
              └───────────┬───────────┘
                          │
                   Aggregate Repository
```

Beispiel Enum:

```text
C# Enum
   │
   ├── Pending = 1
   ├── Confirmed = 2
   └── Shipped = 3
          │
          ▼
Database
┌────┬───────────┬─────────┬────────────────────┐
│ Id │ Name      │ IsFinal │ weitere Metadaten  │
├────┼───────────┼─────────┼────────────────────┤
│ 1  │ Pending   │ false   │ ...                │
│ 2  │ Confirmed │ false   │ ...                │
│ 3  │ Shipped   │ true    │ ...                │
└────┴───────────┴─────────┴────────────────────┘
```

Beispiel Cache:

```text
Request
   │
   ▼
 Cache
   │
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

Beispiel optionales Tenant-Konzept:

```text
Single-Tenant

Application
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
Repository / EF Core
    │
    ▼
Tenant Isolation
    │
    ▼
Database
```

---

# 45. PlantUML-Diagramme

Erstelle mehrere separate PlantUML-Diagramme.

Alle müssen:

- syntaktisch korrekt sein
- direkt kopierbar sein
- verständlich beschriftet sein
- Generics sinnvoll darstellen
- Constraints über Notes oder Beschriftungen darstellen
- Vererbung darstellen
- Interface-Implementierungen darstellen
- Composition verwenden, wo fachlich sinnvoll

Erstelle mindestens:

## 45.1 Entity-Klassendiagramm

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

sowie Generic Constraints.

Markiere Tenant als **optional**.

## 45.2 Aggregate Diagram

```text
Order
 ├── OrderLine
 ├── OrderLine
 └── ShippingAddress
```

## 45.3 Repository Diagram

```text
IOrderRepository
       │
       ▼
     Order
       │
       ├── OrderLine
       └── ShippingAddress
```

Child-Repositories dürfen nicht erscheinen.

## 45.4 Enum Entity Diagram

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
   ├── Metadata
   └── ...
```

sowie:

```text
Order.StatusId
       │
       ▼
OrderStatus.Id
```

## 45.5 Cache Diagram

Zeige:

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
 │  Repository
 │      │
 │      ▼
 │   EF Core
 │      │
 │      ▼
 │  Database
 └──────┘
```

## 45.6 Cache Stampede Diagram

Zeige den Unterschied zwischen:

```text
5 Requests
   │
   ▼
5 Database Queries
```

und:

```text
5 Requests
   │
   ▼
1 Database Query
   │
   ▼
Cache
   │
   ▼
5 Responses
```

## 45.7 Multi-Instance Cache Diagram

Zeige:

```text
Database
   │
   ├── App A ── Cache A
   ├── App B ── Cache B
   └── App C ── Cache C
```

und:

```text
Database
   │
   ▼
Shared Cache
 ┌──┼──┐
 ▼  ▼  ▼
A   B  C
```

## 45.8 EF Configuration Diagram

Zeige:

```text
EntityConfiguration
       │
       ├── AggregateRootConfiguration
       ├── ChildEntityConfiguration
       ├── EnumEntityConfiguration
       ├── optional Tenant Configuration
       └── optional Audit Configuration
```

## 45.9 Single-Tenant vs. Multi-Tenant Diagram

Erstelle ein Diagramm, das explizit zeigt:

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
                    EF Core
                       │
                       ▼
                   Database
```

und optional:

```text
                    Domain
                       │
                       ▼
                   Aggregate
                       │
                       ▼
                  Tenant-aware
                       │
                       ▼
                  Repository
                       │
                       ▼
             Tenant Isolation
                       │
                       ▼
                    EF Core
                       │
                       ▼
                   Database
```

Zeige, dass der zweite Pfad eine Erweiterung des ersten ist.

## 45.10 Repository Architecture Diagram

Zeige die komplette Verbindung:

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
   │
   └── Reference Data
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
```

---

# 46. Architektonische Bewertung

Bewerte ausdrücklich:

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
11. Audit
12. SaveChanges Interceptor
13. Strongly Typed IDs
14. Entity Equality
15. EF-Core-Proxies
16. Persistierbare Enum Entities
17. C# Enum vs. Smart Enum
18. Enum Seed Data
19. numerischer Enum-Wert als Primary Key
20. String-Wert
21. zusätzliche Enum-Metadaten
22. Reference Data Repository
23. Cache Repository
24. Cache Decorator
25. Cache-aside
26. Read-through
27. Lazy Loading
28. Startup Preloading
29. Cache Stampede
30. Cache Invalidierung
31. IMemoryCache
32. Distributed Cache
33. Redis
34. Repository vs. Query Handler
35. CQRS
36. Specification Pattern
37. Unit of Work
38. DbContext als Unit of Work
39. Testbarkeit
40. Performance
41. Skalierbarkeit
42. Multi-Instance Deployment
43. Single-Tenant Deployment
44. optionale Tenant Capability

---

# 47. Besonders wichtig: Repository vs. technische Abstraktion

Die Antwort soll ausdrücklich kritisch hinterfragen, ob folgende Architektur sinnvoll wäre:

```text
IRepository
    ├── ICacheRepository
    ├── ITenantRepository
    ├── IAuditRepository
    ├── IEnumRepository
    ├── IAggregateRepository
    └── IChildEntityRepository
```

Vermeide diese Struktur, wenn sie lediglich unterschiedliche technische Verantwortlichkeiten unter einem Begriff zusammenfasst.

Eine mögliche bessere Trennung könnte sein:

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
Infrastructure / EF Core Interceptor
```

Dies ist nur eine Hypothese.

Bewerte sie und entscheide selbst.

Die finale Antwort soll eindeutig sagen:

> Welche Abstraktionen sind tatsächlich Repositorys und welche sollten besser als Service, Decorator, Context oder Infrastructure Component modelliert werden?

Dabei muss ausdrücklich unterschieden werden zwischen:

```text
Basisarchitektur
```

und:

```text
optionalen Capabilities
```

insbesondere:

```text
Multi-Tenancy
Audit
Caching
```

---

# 48. Repository Lifetimes

Untersuche:

```text
Singleton
Scoped
Transient
```

in Bezug auf:

```text
DbContext
Repository
Cache
ReferenceDataService
TenantContext
```

Zeige konkrete DI-Registrierungen.

Berücksichtige insbesondere:

```text
DbContext = Scoped
Repository = Scoped
```

und entscheide, ob ein Reference-Data-Cache:

```text
Singleton
```

sein sollte.

Berücksichtige bei Singleton-Caches:

- thread safety
- memory usage
- Application Lifetime
- multiple instances
- cache invalidation

Zeige ausdrücklich, welche Services in einer Single-Tenant-Anwendung gar nicht registriert werden müssen.

Beispielsweise:

```text
Single-Tenant:
ITenantContext → nicht registriert

Multi-Tenant:
ITenantContext → registriert
```

---

# 49. Testbarkeit

Zeige, wie die Architektur getestet werden kann.

Mindestens:

## Domain Tests

```text
Order
OrderLine
Aggregate Invariants
Equality
```

## Repository Tests

```text
OrderRepository
ReferenceDataRepository
```

## Cache Tests

Teste insbesondere:

```text
Cache Hit
Cache Miss
Concurrent Cache Miss
Cache Invalidation
```

Ein wichtiger Test soll sicherstellen:

```text
5 parallele Requests
        ↓
1 Database Query
        ↓
5 Cache Results
```

## Tenant Tests

Nur wenn Multi-Tenancy aktiviert ist:

```text
Tenant A cannot access Tenant B
```

## Integration Tests

Berücksichtige:

```text
EF Core
Database
Tenant Isolation
Seed Data
Enum Consistency
```

---

# 50. Sicherheitsaspekte

Berücksichtige insbesondere:

- Tenant Isolation
- Cross-Tenant Access
- Manipulation von Enum IDs
- Manipulation von Reference Data
- Cache Poisoning
- falsche Cache Keys
- stale Tenant Context
- falsche DI Lifetime
- fehlende Database Constraints

Unterscheide klar zwischen:

```text
Single-Tenant Security
```

und:

```text
Multi-Tenant Security
```

Erkläre, welche Sicherheitsmaßnahmen auf:

```text
Application
Repository
EF Core
Database
```

umgesetzt werden sollten.

---

# 51. Vollständiger Beispielcode

Liefere vollständigen, möglichst produktionsnahen C#-Code für die finale Architektur.

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

sofern diese tatsächlich in der finalen Architektur benötigt werden.

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

Falls empfohlen:

```text
ITenantRepository<...>
TenantRepository<...>
```

Falls nicht empfohlen, erkläre warum.

## Cache

```text
IReferenceDataCache<T,TKey>
ReferenceDataCache<T,TKey>

CacheKeyFactory

ReferenceDataService
```

sowie den Mechanismus für Concurrent Cache Misses.

## EF Core

```text
AppDbContext

EntityConfiguration
AggregateRootConfiguration
ChildEntityConfiguration
EnumEntityConfiguration
TenantConfiguration
AuditConfiguration

OrderConfiguration
OrderLineConfiguration
ShippingAddressConfiguration
OrderStatusConfiguration
```

nur soweit diese Klassen in der finalen Architektur tatsächlich benötigt werden.

## Audit

Zeige:

```text
SaveChangesInterceptor
ICurrentUser
ITimeProvider
```

oder die tatsächlich empfohlene Lösung.

## Application

Zeige mindestens einen Application Service / Command Handler, der:

```text
OrderRepository
```

verwendet und gleichzeitig Reference Data über den vorgesehenen Cache-Zugriff bezieht.

Zeige außerdem, wie derselbe Code in:

```text
Single-Tenant
```

und:

```text
Multi-Tenant
```

funktioniert.

Der Single-Tenant-Code soll nicht mit unnötigen Tenant-Abhängigkeiten belastet werden.

---

# 52. Architekturgrenzen

Achte auf eine klare Trennung:

```text
┌───────────────────────────────────────┐
│               Domain                  │
│                                       │
│ Entities                              │
│ Aggregates                            │
│ Value Objects                         │
│ Domain Rules                          │
│ Interfaces                            │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│             Application               │
│                                       │
│ Use Cases                             │
│ Commands                              │
│ Queries                               │
│ Services                              │
│ Repository Interfaces                 │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│           Infrastructure              │
│                                       │
│ EF Core                               │
│ Repositories                          │
│ Database                              │
│ Cache                                 │
│ Redis                                 │
│ Interceptors                          │
│ optional Tenant Infrastructure        │
└───────────────────────────────────────┘
```

Erkläre, in welcher Schicht welche Abstraktion liegen sollte.

Besonders wichtig:

Eine Single-Tenant-Domain soll nicht von Infrastrukturklassen wie:

```text
TenantContext
Redis
EF Core
```

abhängen.

---

# 53. Finale Gesamtarchitektur

Am Ende soll eine klare Gesamtarchitektur entstehen.

Für Single-Tenant beispielsweise:

```text
                         APPLICATION
                              │
               ┌──────────────┴──────────────┐
               │                             │
           Commands                       Queries
               │                             │
               ▼                             ▼
       Aggregate Repository            Query Handler
               │                             │
               ▼                             ├── EF Core
          Aggregate                         └── Dapper
               │
               ▼
           DbContext
               │
               ▼
            Database


          REFERENCE DATA
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

Optional für Multi-Tenancy:

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
Tenant-aware Repository / EF Core
   │
   ▼
Tenant Isolation
   │
   ▼
Database
```

Zeige klar, dass diese Tenant-Schicht **additiv** ist.

---

# 54. Wichtige Abschlussanforderung

Am Ende möchte ich nicht nur eine Sammlung von Klassen sehen, sondern eine **begründete Architekturentscheidung**.

Beantworte insbesondere diese Fragen eindeutig:

## Entities

- Wie sieht die endgültige Entity-Basisklasse aus?
- Welche Interfaces gibt es?
- Welche Generic Constraints gibt es?

## Aggregate

- Wie wird ein Aggregate Root gekennzeichnet?
- Wie werden Child Entities gekennzeichnet?
- Wie wird verhindert, dass Child Entities eigene Repositories bekommen?

## Tenant

- Ist Tenant Bestandteil der Basisarchitektur?
- Wie wird Tenant optional aktiviert?
- Gibt es ein `TenantRepository`?
- Oder ist `ITenantContext` + Query Filter besser?
- Wo findet Tenant Isolation statt?
- Wie funktioniert dieselbe Architektur ohne Tenant?

## Audit

- Ist Audit Teil der Entity-Vererbung?
- Oder wird es über Interface + Infrastructure umgesetzt?
- Wie werden Audit-Werte automatisch gesetzt?

## Enum Entities

- Was genau ist eine Enum Entity?
- Wie wird der numerische Enum-Wert zum Primary Key?
- Wie wird der String-Wert gespeichert?
- Wie werden zusätzliche Metadaten gespeichert?
- Wie wird die Konsistenz mit dem C# Enum sichergestellt?
- Wie werden die Werte geseedet?
- Warum ist dies keine normale Lookup Entity?

## Repository

- Was ist das generische Repository?
- Was ist ein Aggregate Repository?
- Welche konkreten Repositorys gibt es?
- Gibt es ein Tenant Repository?
- Gibt es ein Reference Data Repository?
- Gibt es ein Cache Repository?
- Wenn nein: Warum nicht?

## Cache

- Wo befindet sich der Cache?
- Wie funktioniert Cache-aside?
- Was passiert beim ersten Zugriff?
- Was passiert bei weiteren Zugriffen?
- Wie wird Cache Stampede verhindert?
- Wie wird invalidiert?
- Wie funktioniert das mit mehreren Application-Instanzen?

## EF Core

- Wie sehen die Configurations aus?
- Wie werden Strongly Typed IDs gespeichert?
- Wie werden Enum Entities konfiguriert?
- Wie werden Aggregate und Child Entities konfiguriert?
- Wie funktionieren Tenant Query Filters?
- Wie werden Tenant Query Filters in einer Single-Tenant-Anwendung vollständig vermieden?

## Architektur

- Welche Abstraktionen gehören in Domain?
- Welche in Application?
- Welche in Infrastructure?
- Wo befindet sich EF Core?
- Wo befindet sich Redis?
- Wo befindet sich der Cache?
- Welche Komponenten sind optional?
- Welche Komponenten sind zwingender Bestandteil der Basisarchitektur?

---

# 55. Entscheidende Vergleichsdarstellung

Erstelle am Ende eine Tabelle nach folgendem Prinzip:

| Capability | Single-Tenant | Multi-Tenant |
|---|---|---|
| Entity | Ja | Ja |
| Aggregate Root | Ja | Ja |
| Child Entity | Ja | Ja |
| Repository | Ja | Ja |
| Audit | optional | optional |
| Tenant Context | Nein | Ja |
| Tenant Entity Interface | Nein | optional |
| Tenant Query Filter | Nein | Ja |
| Tenant Repository | möglichst Nein | nur falls begründet |
| Reference Data Cache | optional | optional |
| Distributed Cache | abhängig von Deployment | abhängig von Deployment |

Korrigiere diese Tabelle, falls deine Architektur zu einem anderen Ergebnis kommt.

Das Ziel ist, klar zu zeigen:

> **Multi-Tenancy erweitert die Architektur, anstatt die Basisarchitektur zu definieren.**

---

# 56. Stil der Antwort

Erkläre wichtige Architekturentscheidungen **direkt an der Stelle, an der sie relevant sind**.

Wenn beispielsweise ein PlantUML-Diagramm eine wichtige Beziehung zeigt, erkläre unmittelbar danach:

- warum diese Beziehung existiert
- welche Alternative es gibt
- warum die gewählte Variante besser ist

Verwende:

- Markdown
- Tabellen
- Textdiagramme
- PlantUML
- C# Codeblöcke

in sinnvoller Kombination.

Verwende PlantUML für strukturelle Zusammenhänge und Textdiagramme für einfache Ablaufdarstellungen.

Die PlantUML- und C#-Blöcke sollen **direkt kopierbar** sein.

Der C#-Code soll untereinander konsistent sein.

Vermeide Pseudocode, wenn echter C#-Code sinnvoll möglich ist.

Wenn du eine Entscheidung triffst, kennzeichne sie beispielsweise mit:

> **Empfehlung**

und erkläre anschließend kurz die Begründung.

Die Antwort darf umfangreich sein. Qualität, Konsistenz und architektonische Begründung sind wichtiger als Kürze.

Die finale Empfehlung soll eine konkrete, zusammenhängende Architektur für eine produktive ASP.NET-Core-/EF-Core-Anwendung darstellen.

Die wichtigste architektonische Eigenschaft ist dabei:

```text
                    ┌───────────────────┐
                    │  Basisarchitektur │
                    │                   │
                    │ Entity            │
                    │ Aggregate         │
                    │ Repository        │
                    │ EF Core           │
                    └─────────┬─────────┘
                              │
                    optionale Capabilities
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
            Audit        Multi-Tenancy       Cache
```

Die Basisarchitektur muss vollständig ohne Multi-Tenancy funktionieren.

Eine Multi-Tenant-Anwendung soll dieselbe Basisarchitektur verwenden können und lediglich die dafür notwendigen zusätzlichen Komponenten aktivieren.

Die Gesamtstruktur soll insbesondere diese Kette nachvollziehbar abbilden:

```text
Entity
   ↓
Aggregate Root / Child Entity
   ↓
optional Tenant / Audit
   ↓
Enum Entity / Reference Data
   ↓
Repository
   ↓
Application Service / Command
   ↓
Cache
   ↓
EF Core
   ↓
Database
```

Dabei soll klar erkennbar sein, **welche dieser Komponenten tatsächlich miteinander verbunden sind und welche bewusst voneinander getrennt bleiben**.

Besonders wichtig ist außerdem die klare Unterscheidung:

```text
Single-Tenant
```

ist kein Sonderfall einer Multi-Tenant-Architektur, sondern soll ein **vollwertiger, möglichst einfacher Anwendungsfall der Basisarchitektur** sein.

```text
Multi-Tenant
```

ist eine **optionale Erweiterung**, die nur dort sichtbar wird, wo sie tatsächlich benötigt wird.