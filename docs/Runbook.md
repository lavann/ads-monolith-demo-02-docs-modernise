# Runbook - Retail Monolith

## Overview
Operational guide for building, running, and troubleshooting the Retail Monolith application.

## Prerequisites

### Required Software
- **.NET 8.0 SDK** or later
  - Download: https://dotnet.microsoft.com/download/dotnet/8.0
  - Verify: `dotnet --version` (should show 8.x or higher)

- **SQL Server LocalDB** (for Windows development)
  - Included with Visual Studio
  - Alternative: Full SQL Server (Express or Developer edition)
  - Alternative: SQL Server in Docker

- **Git** (for cloning repository)

### Optional Tools
- **Visual Studio 2022** (v17.8+) or **Visual Studio Code** with C# extension
- **Azure Data Studio** or **SQL Server Management Studio** (for database inspection)
- **EF Core CLI Tools** (for manual migrations)
  ```bash
  dotnet tool install --global dotnet-ef
  ```

## Quick Start

### 1. Clone Repository
```bash
git clone https://github.com/lavann/ads-monolith-demo-02-docs-modernise.git
cd ads-monolith-demo-02-docs-modernise
```

### 2. Restore Dependencies
```bash
dotnet restore
```

### 3. Run Application
```bash
dotnet run
```

On first run:
- Database will be created automatically (LocalDB or configured SQL Server)
- Migrations will be applied
- 50 sample products will be seeded
- Application will start listening on `http://localhost:5000` and `https://localhost:5001`

### 4. Access Application
- **Home**: http://localhost:5000
- **Products**: http://localhost:5000/Products
- **Cart**: http://localhost:5000/Cart
- **Health Check**: http://localhost:5000/health

## Build Commands

### Standard Build
```bash
dotnet build
```

### Release Build
```bash
dotnet build -c Release
```

### Clean Build
```bash
dotnet clean
dotnet build
```

### Restore NuGet Packages
```bash
dotnet restore
```

## Run Commands

### Run with Development Settings
```bash
dotnet run
```
Uses `appsettings.Development.json` overrides.

### Run with Production Settings
```bash
dotnet run --environment Production
```

### Run with Specific URLs
```bash
dotnet run --urls "http://localhost:8080;https://localhost:8443"
```

### Watch Mode (Auto-Reload)
```bash
dotnet watch run
```
Automatically restarts on code changes.

## Database Commands

### View Current Migrations
```bash
dotnet ef migrations list
```

### Apply Migrations Manually
```bash
dotnet ef database update
```

### Create New Migration
After modifying model classes:
```bash
dotnet ef migrations add <MigrationName>
dotnet ef database update
```

Example:
```bash
dotnet ef migrations add AddProductReviews
dotnet ef database update
```

### Reset Database (Drop and Recreate)
```bash
dotnet ef database drop --force
dotnet run
```
⚠️ **Warning**: This deletes all data!

### Generate SQL Script from Migrations
```bash
dotnet ef migrations script -o migration.sql
```
Useful for reviewing changes or applying migrations manually.

### View Connection String
Connection string is configured in:
- `appsettings.json` (or `appsettings.Development.json`)
- Environment variable: `ConnectionStrings__DefaultConnection`

Default LocalDB connection:
```
Server=(localdb)\MSSQLLocalDB;Database=RetailMonolith;Trusted_Connection=True;MultipleActiveResultSets=true
```

## Testing the Application

### Test Product Listing
```bash
curl http://localhost:5000/Products
```

### Test Adding to Cart (via form POST)
From browser: Navigate to `/Products` and click "Add to Cart" button

### Test Checkout API
```bash
curl -X POST http://localhost:5000/api/checkout \
  -H "Content-Type: application/json"
```
⚠️ Note: Uses hardcoded "guest" customer and "tok_test" payment token

### Test Get Order API
```bash
curl http://localhost:5000/api/orders/1
```

### Test Health Check
```bash
curl http://localhost:5000/health
```
Should return `Healthy` status.

## Configuration

### Connection String Override (Environment Variable)
**Windows (PowerShell):**
```powershell
$env:ConnectionStrings__DefaultConnection = "Server=YOUR_SERVER;Database=RetailMonolith;..."
dotnet run
```

**Linux/Mac:**
```bash
export ConnectionStrings__DefaultConnection="Server=YOUR_SERVER;Database=RetailMonolith;..."
dotnet run
```

### Environment Override
```bash
export ASPNETCORE_ENVIRONMENT=Production
dotnet run
```

## Common Issues and Solutions

### Issue 1: "LocalDB is not installed or not found"
**Symptoms:**
- Error: `A network-related or instance-specific error occurred while establishing a connection to SQL Server`
- LocalDB not available

**Solutions:**
1. Install SQL Server LocalDB (comes with Visual Studio)
2. Use SQL Server Express or Developer Edition
3. Use SQL Server in Docker:
   ```bash
   docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=YourStrong@Passw0rd" \
     -p 1433:1433 --name sqlserver \
     -d mcr.microsoft.com/mssql/server:2022-latest
   ```
4. Update connection string in `appsettings.json`:
   ```json
   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=localhost,1433;Database=RetailMonolith;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=True"
     }
   }
   ```

### Issue 2: "Database already exists" or Schema Out of Sync
**Symptoms:**
- Migration errors
- Entity/table mismatches
- Foreign key violations

**Solution:**
```bash
dotnet ef database drop --force
dotnet run
```
This recreates the database from scratch.

### Issue 3: Application Hangs on Startup
**Symptoms:**
- Application starts but doesn't respond
- Timeout on first request

**Likely Cause:**
- Auto-migration is running (could take 10-30 seconds on first run)
- Database connection timeout

**Solutions:**
1. Wait for migration to complete (check terminal output)
2. Check database connectivity
3. Review connection string configuration

### Issue 4: Duplicate Cart Items Added
**Symptoms:**
- Clicking "Add to Cart" adds duplicate entries instead of incrementing quantity

**Root Cause:**
- Known issue in `Pages/Products/Index.cshtml.cs` (lines 27-50)
- Duplicate cart logic: both direct DbContext manipulation and CartService call

**Workaround:**
- This is documented tech debt; functionality still works but creates unnecessary database entries

### Issue 5: Port Already in Use
**Symptoms:**
- Error: `Unable to bind to http://localhost:5000 on the IPv4 loopback interface: 'Address already in use'`

**Solutions:**
1. Change port:
   ```bash
   dotnet run --urls "http://localhost:5050"
   ```
2. Find and kill process using port 5000:
   **Windows:**
   ```powershell
   netstat -ano | findstr :5000
   taskkill /PID <PID> /F
   ```
   **Linux/Mac:**
   ```bash
   lsof -ti:5000 | xargs kill
   ```

### Issue 6: HTTPS Certificate Issues
**Symptoms:**
- Browser shows SSL/TLS errors
- "Your connection is not private" warning

**Solution:**
Trust the development certificate:
```bash
dotnet dev-certs https --trust
```

### Issue 7: "Products table is empty" After Manual Database Changes
**Symptoms:**
- No products showing on `/Products` page after manual database operations

**Solution:**
Trigger re-seeding by dropping Products table:
```sql
DELETE FROM Products;
```
Then restart application (auto-seeding will run).

## Known Issues and Technical Debt

### 1. Hardcoded "guest" Customer ID
**Impact**: All users share the same cart
**Location**: Throughout application (services, page models, API endpoints)
**Workaround**: None; single-user development mode
**Future**: Requires authentication/authorization implementation

### 2. Duplicate Cart Logic in Products Page
**Impact**: Inefficient, potential for bugs
**Location**: `Pages/Products/Index.cshtml.cs` OnPostAsync method
**Workaround**: Functionality works despite redundancy
**Future**: Refactor to use only CartService

### 3. No Inventory Validation in Add to Cart
**Impact**: Users can add more items than available stock
**Location**: `CartService.AddToCartAsync()`
**Workaround**: Validation happens at checkout; user experience is poor
**Future**: Add inventory check in cart service

### 4. Inventory Not Rolled Back on Payment Failure
**Impact**: Failed payments leave inventory reserved
**Location**: `CheckoutService.CheckoutAsync()`
**Workaround**: Mock payment gateway always succeeds (hides issue)
**Future**: Implement compensation logic or use reservation pattern

### 5. Auto-Migration at Startup
**Impact**: Startup delay, not production-ready
**Location**: `Program.cs` lines 24-29
**Workaround**: Acceptable for development/demo
**Future**: Use pre-deployment migration strategy for production (see ADR-004)

### 6. Mock Payment Gateway
**Impact**: Not production-ready
**Location**: `MockPaymentGateway.cs`
**Workaround**: Suitable for development/testing
**Future**: Implement real payment gateway integration (see ADR-003)

### 7. Incomplete Checkout and Orders UI
**Impact**: Limited user-facing functionality
**Location**: `Pages/Checkout/Index.cshtml.cs`, `Pages/Orders/Index.cshtml.cs`
**Workaround**: Use API endpoints directly
**Future**: Implement full Razor Pages UI

### 8. No Logging or Observability
**Impact**: Hard to troubleshoot production issues
**Location**: Throughout application
**Workaround**: Use debugger or database inspection
**Future**: Add structured logging (Serilog), telemetry (Application Insights)

### 9. No Unit Tests
**Impact**: Refactoring risk, regression potential
**Location**: No test project exists
**Workaround**: Manual testing
**Future**: Add xUnit test project with service layer tests

### 10. Health Check Not Comprehensive
**Impact**: Health endpoint doesn't verify database connectivity
**Location**: `Program.cs` health check registration
**Workaround**: Basic health check sufficient for current needs
**Future**: Add database health check, external dependency checks

## Performance Considerations

### Database Queries
- **N+1 Query Risk**: Always use `.Include()` for navigation properties
- **Example**: `db.Carts.Include(c => c.Lines)` in CartService
- **Monitoring**: Enable EF Core query logging in development:
  ```json
  {
    "Logging": {
      "LogLevel": {
        "Microsoft.EntityFrameworkCore.Database.Command": "Information"
      }
    }
  }
  ```

### Connection Pooling
- EF Core uses connection pooling by default
- `MultipleActiveResultSets=true` in connection string enables MARS
- Max pool size: 100 (default)

## Deployment Notes

### Azure App Service
1. Set connection string in Azure Portal (Configuration > Connection Strings)
2. Ensure SQL Server firewall allows Azure Services
3. Consider disabling auto-migration for production (see ADR-004)

### Docker (Future)
Not currently containerized, but would require:
- Dockerfile with .NET 8 runtime
- Connection string environment variable
- External SQL Server (not LocalDB)

## Support and Resources

### Documentation
- High-Level Design: `/docs/HLD.md`
- Low-Level Design: `/docs/LLD.md`
- Architecture Decision Records: `/docs/ADR/`

### Related Projects
- Original repository: https://github.com/lavann/ads_monotlith_app

### EF Core Resources
- [EF Core Documentation](https://docs.microsoft.com/en-us/ef/core/)
- [ASP.NET Core Documentation](https://docs.microsoft.com/en-us/aspnet/core/)
