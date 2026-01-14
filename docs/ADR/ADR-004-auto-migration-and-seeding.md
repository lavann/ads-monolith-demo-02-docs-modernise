# ADR-004: Auto-Migration and Seeding at Application Startup

## Status
Accepted (with caveats for production)

## Context
The application uses Entity Framework Core with Code-First migrations. A decision was needed on when and how to apply database migrations and seed initial data.

## Decision
Automatically apply pending EF Core migrations and seed sample data during application startup, before the first HTTP request is handled.

## Implementation

In `Program.cs`, after building the application but before calling `app.Run()`:

```csharp
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
    await AppDbContext.SeedAsync(db); // seed the database
}
```

### Migration
`db.Database.MigrateAsync()` applies all pending migrations found in the `Migrations/` folder to bring the database schema up to date with the current model.

### Seeding
`AppDbContext.SeedAsync(db)` checks if the Products table is empty and, if so:
- Generates 50 sample products with random categories, names, and prices (£5-£105)
- Creates corresponding inventory records with random stock (10-200 units)
- Categories: Apparel, Footwear, Accessories, Electronics, Home, Beauty

## Rationale

### Why Auto-Migration at Startup
1. **Developer Convenience**: No manual `dotnet ef database update` step
2. **Fresh Start**: Each deployment or restart ensures schema is current
3. **Simplified Onboarding**: New developers don't need to know EF tooling
4. **CI/CD Simplicity**: Deployments don't need separate migration step
5. **Matches Demo/Hackathon Needs**: Quick setup for evaluation or prototyping

### Why Startup Seeding
1. **Self-Contained**: Application runs without manual data setup
2. **Reproducible**: Same seed data across all environments
3. **Demo-Ready**: Immediate usable product catalog for testing
4. **Idempotent**: Only seeds if database is empty (checks `Products.AnyAsync()`)

## Consequences

### Positive
- **Zero-Friction Setup**: Clone repo, run app, it works
- **No Database Scripts**: No separate seed SQL files to maintain
- **Consistent Test Data**: Deterministic seed logic (except random values)
- **Fast Iteration**: Schema changes automatically applied on restart

### Negative

#### Production Risks
1. **Startup Delay**: First request waits for migration + seeding
   - Could timeout health checks
   - Cold start penalty in serverless/container environments
   
2. **Migration Failures Block Startup**: If migration fails, application won't start
   - No graceful degradation
   - Requires manual intervention to fix
   
3. **Concurrent Deployments**: Multiple instances migrating simultaneously
   - Race conditions possible
   - EF Core uses `__EFMigrationsHistory` table with database locks for coordination
   - Protection mechanism: First instance acquires lock, others wait
   - Limitations: Lock contention can cause timeouts; doesn't prevent schema incompatibility between running app versions
   - Blue-green deployments could have schema incompatibility if old and new code run simultaneously
   
4. **Breaking Migrations**: Destructive migrations (drop column) applied automatically
   - No review or approval gate
   - Cannot easily roll back
   - Downtime risk if migration requires long table locks

5. **No Migration Control**: Cannot apply migrations in stages
   - All-or-nothing application
   - Cannot skip or reorder migrations dynamically

#### Development Concerns
1. **Hidden Dependency**: Startup code not obvious to newcomers
   - Could confuse developers expecting manual migrations
   
2. **Seed Data in Production**: Seeding logic runs in all environments
   - Could accidentally seed production database
   - Current implementation checks for empty table (mitigation)
   
3. **Connection String Required**: Application cannot start without valid database
   - Cannot run without database even for static file serving
   - Complicates disconnected development

## Best Practices for Production

### Should NOT Use Auto-Migration If:
- Running multiple instances (horizontal scaling)
- Zero-downtime deployments required
- Schema changes need approval/review process
- Rollback strategy needed
- Database managed by separate DBA team

### Should Use Auto-Migration If:
- Single instance deployment
- Development/staging environments
- Hackathon/demo/prototype
- Small team with full control
- Acceptable downtime during deployments

### Recommended Production Approach
Instead of auto-migration at startup:
1. Apply migrations in CI/CD pipeline before deployment
2. Use EF Core bundle or SQL scripts: `dotnet ef migrations script`
3. Run migrations as separate deployment step
4. Use migration jobs/init containers in Kubernetes
5. Consider database migration tools (FluentMigrator, DbUp)

### Seeding in Production
- Remove auto-seeding or guard with environment check:
  ```csharp
  if (!app.Environment.IsProduction())
  {
      await AppDbContext.SeedAsync(db);
  }
  ```
- Use separate data import process for production data
- Consider feature flags to enable/disable seeding

## Current Assessment

### Appropriate For This Application?
**Yes**, because:
- Demo/hackathon application (per README)
- Comment in code: "auto-migrate & seed (hack convenience)"
- Designed for local development and evaluation
- Single instance deployment assumed

### Production Readiness
**No**, would need:
- Environment-based migration strategy
- Separate seeding for production
- Health check that doesn't depend on first request
- Logging/monitoring of migration execution
- Rollback plan for failed migrations

## Code Comments
The code itself acknowledges this is a "hack convenience":
```csharp
// auto-migrate & seed (hack convenience)
```

This indicates awareness that this is appropriate for current context but not production-grade.

## Alternatives Considered

### Manual Migration Step
- **Pros**: Full control, explicit, safer for production
- **Cons**: Extra step, requires EF tooling, can be forgotten
- **Verdict**: More appropriate for production, but overkill for demo

### Startup Migration + Environment Check
- **Pros**: Convenience in dev, safety in production
- **Cons**: Still has startup delay in dev
- **Verdict**: Good compromise; recommend adding

### Database Initializer Pattern
- **Pros**: Explicit initialization, can be called separately
- **Cons**: Same risks as current approach
- **Verdict**: No significant benefit over current approach

### Init Container (Kubernetes)
- **Pros**: Separate migration from app startup
- **Cons**: Requires orchestration platform
- **Verdict**: Not applicable to current deployment model

## Related Decisions
- ADR-001: Monolithic architecture (single database, simpler migration)
- ADR-002: Entity Framework Core (provides migration infrastructure)

## Future Improvements
1. Add environment check to skip seeding in production
2. Add structured logging for migration events
3. Add telemetry for migration duration
4. Consider health check endpoint that reports migration status
5. Document migration strategy in deployment guide
