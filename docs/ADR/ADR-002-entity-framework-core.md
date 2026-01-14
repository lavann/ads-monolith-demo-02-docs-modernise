# ADR-002: Entity Framework Core with SQL Server

## Status
Accepted

## Context
The application needs to persist product catalog, inventory, shopping carts, and orders. A decision was needed on the data access technology and database platform to use.

## Decision
Use Entity Framework Core 9.0.9 as the ORM with SQL Server as the database engine. Use Code-First approach with migrations.

## Rationale

### Why Entity Framework Core
1. **Native .NET Integration**: First-class citizen in .NET/ASP.NET Core ecosystem
2. **Code-First**: Models defined in C# with automatic schema generation
3. **LINQ Support**: Type-safe queries with IntelliSense and compile-time checking
4. **Change Tracking**: Automatic detection of entity state changes
5. **Migrations**: Version-controlled schema evolution with `dotnet ef` tooling
6. **Convention over Configuration**: Minimal configuration needed for basic scenarios

### Why SQL Server
1. **Relational Model**: Natural fit for normalized data (Products, Carts, Orders)
2. **Transactions**: ACID guarantees for checkout operations
3. **Mature Tooling**: Well-supported in Azure and on-premises
4. **LocalDB**: Easy development experience without full SQL Server installation
5. **Team Familiarity**: Common choice in .NET ecosystem

### Code-First Approach
- Models defined in `Models/` namespace as C# classes
- `AppDbContext` manages DbSets and configuration
- Migrations generated with `dotnet ef migrations add`
- Schema applied with `dotnet ef database update` or auto-migration at runtime

## Implementation Details

### DbContext Configuration
```csharp
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### Connection Strings
- **Development**: LocalDB `(localdb)\\MSSQLLocalDB`
- **Production**: Configurable via `appsettings.json` or environment variable
- **Design-Time**: Hardcoded in `DesignTimeDbContextFactory` for migrations

### Entity Relationships
- One-to-Many: Cart → CartLines
- One-to-Many: Order → OrderLines
- Unique Indexes: Product.Sku, InventoryItem.Sku

### Migration Strategy
- Migrations stored in `/Migrations` directory
- Initial migration creates all tables
- Auto-migration at startup for convenience (see ADR-004)

## Consequences

### Positive
- Rapid development with minimal boilerplate
- Type-safe database access reduces runtime errors
- Automatic SQL generation based on LINQ queries
- Migration history tracked in source control
- Strong tooling support (Visual Studio, Rider, VS Code)
- Easy to add new entities and relationships

### Negative
- **SQL Server Dependency**: Application tightly coupled to SQL Server
  - Cannot easily switch to PostgreSQL, MySQL, or NoSQL
  - LocalDB requirement may complicate CI/CD or containerization
- **EF Overhead**: Performance overhead compared to raw ADO.NET or Dapper
  - N+1 query problems possible without careful use of `.Include()`
- **Migration Complexity**: Schema changes require careful migration planning
- **Abstraction Leakage**: EF-specific patterns leak into service layer
  - Services directly use `DbContext` and `.Include()`, `.AsNoTracking()`, etc.
  - No repository pattern to abstract data access
- **Connection Pooling**: Multiple active result sets required (MARS)

### Known Issues
1. **Hardcoded Connection String**: `DesignTimeDbContextFactory` has hardcoded LocalDB connection
   - Maintenance burden if connection string changes
   - Cannot run migrations against different environment easily

2. **Direct DbContext in Services**: No abstraction layer
   - Tight coupling makes unit testing difficult
   - Requires database for integration tests

3. **Lazy Loading Disabled**: Manual `.Include()` required
   - Easy to miss, causing N+1 queries
   - Example: `CheckoutService` includes cart lines explicitly

## Alternatives Considered

### Dapper (Micro-ORM)
- **Pros**: Better performance, more control over SQL
- **Cons**: More boilerplate, manual mapping, no change tracking
- **Verdict**: Premature optimization; EF Core sufficient for current needs

### NoSQL (MongoDB, CosmosDB)
- **Pros**: Flexible schema, horizontal scaling
- **Cons**: Loss of ACID transactions, less suitable for relational data
- **Verdict**: Relational model fits domain well; SQL Server appropriate

### ADO.NET
- **Pros**: Maximum performance, full control
- **Cons**: Significant boilerplate, manual SQL, error-prone
- **Verdict**: Development speed prioritized; EF Core productivity worth overhead

## Related Decisions
- ADR-001: Monolithic architecture (single shared database)
- ADR-004: Auto-migration at startup (EF Core convenience feature)
