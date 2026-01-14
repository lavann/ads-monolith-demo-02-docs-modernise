# Retail Monolith Documentation

This directory contains comprehensive documentation for the Retail Monolith application.

## Documentation Structure

### High-Level Documentation

- **[HLD.md](HLD.md)** - High-Level Design
  - System architecture overview
  - Component and module structure
  - Data stores and external dependencies
  - Runtime assumptions and deployment model

- **[LLD.md](LLD.md)** - Low-Level Design
  - Detailed class and service descriptions
  - Request flows and interactions
  - Areas of coupling and technical hotspots
  - Known technical debt

- **[Runbook.md](Runbook.md)** - Operational Guide
  - Build and run instructions
  - Database management commands
  - Common issues and troubleshooting
  - Configuration and deployment notes

### Architecture Decision Records (ADRs)

The `ADR/` directory contains architecture decisions made in the design of this application:

- **[ADR-001](ADR/ADR-001-monolithic-architecture.md)** - Monolithic Architecture with Logical Domain Separation
  - Why monolithic over microservices
  - Domain boundary identification
  - Future decomposition considerations

- **[ADR-002](ADR/ADR-002-entity-framework-core.md)** - Entity Framework Core with SQL Server
  - ORM and database technology choices
  - Code-First migrations approach
  - Trade-offs and consequences

- **[ADR-003](ADR/ADR-003-mock-payment-gateway.md)** - Mock Payment Gateway for Development
  - Payment processing abstraction
  - Mock implementation for testing
  - Production considerations

- **[ADR-004](ADR/ADR-004-auto-migration-and-seeding.md)** - Auto-Migration and Seeding at Startup
  - Automatic database migration strategy
  - Development convenience vs. production concerns
  - Best practices and alternatives

## Reading Guide

### For New Developers
1. Start with **[HLD.md](HLD.md)** to understand the overall system
2. Read **[Runbook.md](Runbook.md)** to get the application running
3. Review **[LLD.md](LLD.md)** for implementation details
4. Refer to **ADRs** to understand design decisions

### For Architects/Reviewers
1. Review **[HLD.md](HLD.md)** for system design
2. Read all **ADRs** to understand design rationale
3. Check **[LLD.md](LLD.md)** for technical debt and coupling issues

### For Operations/DevOps
1. **[Runbook.md](Runbook.md)** is your primary resource
2. Check **[ADR-004](ADR/ADR-004-auto-migration-and-seeding.md)** for deployment considerations
3. Review **[HLD.md](HLD.md)** for external dependencies

## Key Findings

### Domain Boundaries Identified
- **Products**: Product catalog and metadata
- **Inventory**: Stock level tracking
- **Cart**: Shopping cart management
- **Orders**: Order processing and fulfillment

### Technology Stack
- ASP.NET Core 8.0 (Razor Pages + Minimal APIs)
- Entity Framework Core 9.0.9
- SQL Server / LocalDB
- Mock payment gateway (IPaymentGateway abstraction)

### Known Technical Debt
See **[LLD.md](LLD.md)** section "Areas of Coupling and Hotspots" and "Technical Debt Observations" for detailed list.

Key issues include:
- Hardcoded "guest" customer ID (no authentication)
- Duplicate cart logic in Products page
- No inventory validation in cart operations
- Auto-migration at startup (not production-ready)
- Incomplete UI (Checkout and Orders pages)

## Document Maintenance

These documents reflect the codebase as of their creation date. As the code evolves:

- Update HLD when adding new components or changing architecture
- Update LLD when refactoring services or changing data models
- Create new ADRs for significant design decisions
- Update Runbook with new commands, configurations, or known issues

## Questions or Issues?

If you find inaccuracies or have questions about the documentation, please:
1. Check the actual source code (documentation may be outdated)
2. Review git history for context on changes
3. Raise issues or questions with the development team
