# ADR-001: Monolithic Architecture with Logical Domain Separation

## Status
Accepted

## Context
The application needs to implement e-commerce functionality including product catalog, shopping cart, checkout, and order management. A decision was needed on whether to build this as a distributed system (microservices) or as a single deployable unit (monolith).

## Decision
Build as a single ASP.NET Core monolithic application with logical domain separation into Products, Inventory, Cart, and Orders domains.

## Rationale

### Why Monolithic
1. **Simplicity**: Single deployment unit simplifies development, deployment, and operations
2. **Development Speed**: No distributed system complexity during initial development
3. **Shared Database**: Easier to maintain data consistency with single database and transactions
4. **Team Size**: Suitable for small team or solo development
5. **Refactoring Path**: Code organized by domain enables future decomposition into microservices

### Domain Separation
Even within the monolith, the code is organized around domain boundaries:
- **Products**: Product catalog and metadata
- **Inventory**: Stock level tracking separate from product data
- **Cart**: Shopping cart management per customer
- **Orders**: Order processing and fulfillment

This separation is expressed through:
- Separate model classes for each domain
- Separate service interfaces (ICartService, ICheckoutService)
- Clear dependency directions (Checkout depends on Cart and Inventory)

## Consequences

### Positive
- Fast initial development and deployment
- Easy to reason about data flow and transactions
- Simple debugging and troubleshooting
- Single database transaction spans all operations
- No network latency between components

### Negative
- All domains share the same scaling characteristics (cannot scale independently)
- Deployment is all-or-nothing (no independent deployment of domains)
- Single point of failure (entire application down if it fails)
- All domains coupled to same database technology (SQL Server)
- Potential for domain boundaries to erode over time without discipline

### Future Considerations
The logical domain separation enables future decomposition:
- Each domain could become a microservice
- Comment in CheckoutService mentions future event publishing
- Would require distributed transaction handling (Saga pattern)
- Would need inter-service communication (HTTP/gRPC or messaging)

## Related Decisions
- ADR-002: Entity Framework Core with SQL Server (shared database)
- ADR-004: Auto-migration and seeding at startup (monolith convenience)
