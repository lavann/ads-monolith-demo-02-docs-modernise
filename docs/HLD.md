# High-Level Design (HLD) - Retail Monolith

## Overview

The Retail Monolith is an ASP.NET Core 8 web application that implements a complete e-commerce system in a single deployment unit. It provides product catalog browsing, shopping cart management, checkout functionality, and order processing capabilities.

## Architecture

### Application Type
- **Framework**: ASP.NET Core 8.0
- **UI Pattern**: Razor Pages (server-side rendered)
- **API Pattern**: Minimal APIs for select endpoints
- **Deployment Model**: Single monolithic web application

### Domain Boundaries

While implemented as a monolith, the application has four logical domain areas:

1. **Products Domain**
   - Product catalog management
   - Product information (SKU, name, description, price, category)
   - Active/inactive product status

2. **Inventory Domain**
   - Stock level tracking
   - Inventory reservations during checkout
   - SKU-based inventory management

3. **Cart Domain**
   - Shopping cart management
   - Cart line items (product references, quantities, prices)
   - Cart persistence per customer

4. **Orders Domain**
   - Order creation and management
   - Order line items
   - Order status tracking (Created, Paid, Failed, Shipped)
   - Payment processing integration

## Components and Modules

### Presentation Layer
- **Razor Pages**: Server-side rendered UI
  - `/` - Home page
  - `/Products` - Product listing and "Add to Cart" actions
  - `/Cart` - Shopping cart view
  - `/Checkout` - Checkout page (UI only, minimal implementation)
  - `/Orders` - Orders page (UI only, minimal implementation)
  
- **Minimal APIs**: RESTful endpoints
  - `POST /api/checkout` - Process checkout with payment token
  - `GET /api/orders/{id}` - Retrieve order details
  - `GET /health` - Health check endpoint

### Service Layer

Core business logic is organized into service interfaces and implementations:

- **ICartService / CartService**
  - Get or create cart for customer
  - Add items to cart
  - Get cart with line items
  - Clear cart

- **ICheckoutService / CheckoutService**
  - Orchestrates checkout process
  - Inventory reservation
  - Payment processing
  - Order creation
  - Cart clearing

- **IPaymentGateway / MockPaymentGateway**
  - External payment processing abstraction
  - Currently implemented as a mock for testing

### Data Layer

**Entity Framework Core 9.0.9** with SQL Server provider

**Database Context**: `AppDbContext`
- Manages all entity DbSets
- Configures indexes (unique SKU constraints)
- Provides seeding functionality

## Data Stores

### Primary Database
- **Type**: SQL Server
- **Default**: LocalDB (development)
- **Connection String**: Configurable via `appsettings.json` or environment variable
  ```
  Server=(localdb)\\MSSQLLocalDB;Database=RetailMonolith;Trusted_Connection=True;MultipleActiveResultSets=true
  ```

### Database Tables/Entities

| Table | Purpose | Key Relationships |
|-------|---------|------------------|
| Products | Product catalog | Unique index on SKU |
| Inventory | Stock levels | Unique index on SKU, linked to Products by SKU |
| Carts | Shopping carts | One-to-many with CartLines |
| CartLines | Cart line items | Foreign key to Cart |
| Orders | Customer orders | One-to-many with OrderLines |
| OrderLines | Order line items | Foreign key to Order |

### Data Persistence Strategy
- EF Core migrations for schema management
- Auto-migration on application startup (`db.Database.MigrateAsync()`)
- Auto-seeding of 50 sample products on first run

## External Dependencies

### Direct Dependencies
1. **Microsoft.EntityFrameworkCore.SqlServer** (9.0.9)
   - SQL Server database provider
   
2. **Microsoft.EntityFrameworkCore.Design** (9.0.9)
   - Design-time tooling for migrations
   
3. **Microsoft.AspNetCore.Diagnostics.HealthChecks** (2.2.0)
   - Health check middleware
   
4. **Microsoft.Extensions.Http.Polly** (9.0.9)
   - HTTP resilience and transient fault handling (registered but not actively used)

### External Systems
- **Payment Gateway**: Abstracted via `IPaymentGateway`, currently mocked
  - Real implementation would integrate with external payment processor
  - Currently returns successful mock payment responses

## Runtime Assumptions

### Startup Behavior
1. Application builds services container with:
   - DbContext configured for SQL Server
   - Scoped services: CartService, CheckoutService, PaymentGateway
   - Razor Pages
   - Health checks

2. On first request, auto-migration runs:
   - Applies any pending EF Core migrations
   - Seeds database with 50 sample products if empty
   - Creates inventory records for each product (10-200 units)

3. Application listens on:
   - HTTPS: `https://localhost:5001` (default)
   - HTTP: `http://localhost:5000` (default)

### Environment Modes
- **Development**: No HTTPS redirection enforcement, detailed error pages
- **Production**: HSTS enabled, exception handler middleware active

### Customer Authentication
- **Current State**: All operations use hardcoded "guest" customer ID
- **No Authentication**: No user login, registration, or session management
- **Implication**: Single shared cart across all users (per instance)

### Concurrency and State
- **Database Transactions**: Implicit via EF Core SaveChanges
- **No Distributed State**: All state in SQL Server database
- **Concurrency Risk**: Inventory reservation uses optimistic approach without explicit locking
  - Race conditions possible under high load
  - Inventory could go negative in edge cases

### Scalability Considerations
- Single instance deployment assumed
- No distributed caching or session state
- Database is single point of failure and bottleneck
- Health check endpoint (`/health`) for monitoring readiness

## Configuration

### Configuration Sources
1. `appsettings.json` - Base configuration
2. `appsettings.Development.json` - Development overrides
3. Environment variables - Runtime overrides
4. Connection string: `ConnectionStrings__DefaultConnection`

### Key Settings
- `ConnectionStrings:DefaultConnection` - Database connection string
- `ASPNETCORE_ENVIRONMENT` - Environment mode (Development/Production)

## Deployment Model

- **Type**: Self-contained ASP.NET Core web application
- **Target Runtime**: .NET 8.0
- **Expected Environment**: Windows (LocalDB) or Azure (SQL Server)
- **Build Output**: Single web application binary
- **Dependencies**: Requires SQL Server or LocalDB access
