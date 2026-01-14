# Low-Level Design (LLD) - Retail Monolith

## Overview

This document provides detailed information about the implementation of the Retail Monolith application, including key classes, request flows, and identified areas of technical concern.

## Module Organization

### Namespace Structure

```
RetailMonolith
├── Models/              # Domain entities
├── Data/               # EF Core context and configuration
├── Services/           # Business logic services
├── Pages/              # Razor Pages (UI + Page Models)
├── Migrations/         # EF Core migrations
└── Program.cs          # Application entry point
```

## Key Classes by Domain

### Products Domain

#### `Product` (Models/Product.cs)
Core product entity.

**Properties:**
- `Id` (int) - Primary key
- `Sku` (string) - Stock Keeping Unit, unique identifier
- `Name` (string) - Product name
- `Description` (string?) - Optional description
- `Price` (decimal) - Product price
- `Currency` (string) - Currency code (e.g., "GBP")
- `IsActive` (bool) - Active/inactive status
- `Category` (string?) - Product category

**Usage:**
- Displayed in product listing page
- Referenced when adding items to cart
- No direct service layer; accessed via DbContext

#### `Pages/Products/IndexModel`
Razor Page model for product listing.

**Key Methods:**
- `OnGetAsync()` - Loads all active products from database
- `OnPostAsync(int productId)` - Handles "Add to Cart" form submission

**Issues:**
- Contains duplicate cart logic (both direct DbContext manipulation AND call to CartService)
- Creates in-memory Cart entity that may conflict with database state
- Should delegate fully to CartService

### Inventory Domain

#### `InventoryItem` (Models/InventoryItem.cs)
Tracks stock levels per product SKU.

**Properties:**
- `Id` (int) - Primary key
- `Sku` (string) - Product SKU, unique index
- `Quantity` (int) - Current stock level

**Usage:**
- Created during seeding with random quantities (10-200)
- Decremented during checkout in `CheckoutService`
- No dedicated service layer; accessed directly via DbContext

**Concerns:**
- No locking mechanism for concurrent inventory updates
- Potential for race conditions during high-volume checkouts
- Stock can theoretically go negative if validation timing is poor

### Cart Domain

#### `Cart` (Models/Cart.cs)
Shopping cart entity.

**Properties:**
- `Id` (int) - Primary key
- `CustomerId` (string) - Customer identifier (currently always "guest")
- `Lines` (List<CartLine>) - Collection of cart items

#### `CartLine` (Models/CartLine.cs)
Individual line item in cart.

**Properties:**
- `Id` (int) - Primary key
- `CartId` (int) - Foreign key to Cart
- `Cart` (Cart?) - Navigation property
- `Sku` (string) - Product SKU
- `Name` (string) - Product name (denormalized)
- `UnitPrice` (decimal) - Price per unit (snapshot at add time)
- `Quantity` (int) - Quantity in cart

**Design Note:**
- Denormalizes product name and price for cart stability
- Price stored at add-time, not recalculated from product table

#### `ICartService` / `CartService` (Services/)
Business logic for cart operations.

**Key Methods:**

1. **`GetOrCreateCartAsync(string customerId)`**
   - Retrieves existing cart or creates new one
   - Includes cart lines via `.Include()`
   - Persists new cart immediately

2. **`AddToCartAsync(string customerId, int productId, int quantity)`**
   - Gets or creates cart
   - Validates product exists
   - Updates existing line if SKU already in cart
   - Adds new CartLine if new SKU
   - Saves changes

3. **`GetCartWithLinesAsync(string customerId)`**
   - Returns cart with lines, or new empty cart instance if not found
   - Does not persist empty cart

4. **`ClearCartAsync(string customerId)`**
   - Removes entire cart and associated lines
   - Used after successful checkout

**Issues:**
- No inventory validation during AddToCart
- Allows adding more items than available stock
- Validation only occurs at checkout time

#### `Pages/Cart/IndexModel`
Razor Page model for cart display.

**Key Methods:**
- `OnGetAsync()` - Loads cart and projects to tuples for display

**Design:**
- Uses tuples `(string Name, int Quantity, decimal Price)` for display
- Calculates total in page model
- Read-only; no cart modification on this page

### Orders Domain

#### `Order` (Models/Order.cs)
Customer order entity.

**Properties:**
- `Id` (int) - Primary key
- `CreatedUtc` (DateTime) - Order creation timestamp
- `CustomerId` (string) - Customer identifier
- `Status` (string) - Order status: "Created", "Paid", "Failed", "Shipped"
- `Total` (decimal) - Order total amount
- `Lines` (List<OrderLine>) - Order line items

#### `OrderLine` (Models/OrderLine.cs)
Individual line item in order.

**Properties:**
- `Id` (int) - Primary key
- `OrderId` (int) - Foreign key to Order
- `Order` (Order?) - Navigation property
- `Sku` (string) - Product SKU
- `Name` (string) - Product name (snapshot)
- `UnitPrice` (decimal) - Price per unit (snapshot)
- `Quantity` (int) - Quantity ordered

**Design Note:**
- Immutable snapshot of cart at checkout time
- Price and product details frozen for audit trail

#### `ICheckoutService` / `CheckoutService` (Services/)
Orchestrates the checkout process.

**Key Method: `CheckoutAsync(string customerId, string paymentToken)`**

**Checkout Flow:**
1. **Load Cart**
   - Retrieves cart with `.Include(c => c.Lines)`
   - Throws exception if cart not found

2. **Calculate Total**
   - Sums `UnitPrice * Quantity` for all lines

3. **Reserve Inventory**
   - For each cart line:
     - Fetches inventory by SKU
     - Validates sufficient stock
     - Decrements inventory quantity
   - **Risk**: No transaction isolation; concurrent checkouts could cause issues

4. **Process Payment**
   - Calls `IPaymentGateway.ChargeAsync()`
   - Determines order status based on payment result

5. **Create Order**
   - Creates Order entity
   - Converts CartLines to OrderLines
   - Adds order to DbContext

6. **Clear Cart**
   - Removes all CartLines
   - Leaves empty Cart entity

7. **Save Changes**
   - Commits all changes in single transaction

**Issues:**
- Inventory reservation before payment creates risk of reserved-but-unpaid stock
- If payment fails, inventory is already decremented (not rolled back)
- No compensation logic for payment failures
- Comment indicates future event publishing not implemented

#### `Pages/Checkout/IndexModel`
Minimal implementation; only empty `OnGet()` handler.

**Status**: Checkout UI is not fully implemented; checkout happens via API endpoint.

#### `Pages/Orders/IndexModel`
Minimal implementation; only empty `OnGet()` handler.

**Status**: Orders UI is not fully implemented; order data accessible via API.

### Payment Domain

#### `IPaymentGateway` / `MockPaymentGateway` (Services/)

**Records:**
- `PaymentRequest(decimal Amount, string Currency, string Token)`
- `PaymentResult(bool Succeeded, string? ProviderRef, string? Error)`

**Implementation:**
- `MockPaymentGateway.ChargeAsync()` always returns success
- Generates mock provider reference: `MOCK-{Guid}`
- No actual payment processing

**Future:**
- Interface allows for real payment gateway integration
- Would need error handling, retries, idempotency

## Data Access Layer

### `AppDbContext` (Data/AppDbContext.cs)

**DbSets:**
- `Products` - Product catalog
- `Inventory` - Stock levels
- `Carts` - Shopping carts
- `CartLines` - Cart line items
- `Orders` - Orders
- `OrderLines` - Order line items

**Configuration (OnModelCreating):**
- Unique index on `Product.Sku`
- Unique index on `InventoryItem.Sku`

**Seeding (SeedAsync):**
- Checks if Products table is empty
- Generates 50 products with random categories and prices
- Creates corresponding inventory items with random quantities (10-200)
- Categories: Apparel, Footwear, Accessories, Electronics, Home, Beauty
- Price range: £5-£105

### `DesignTimeDbContextFactory` (Data/DesignTimeDbContextFactory.cs)

**Purpose:**
- Enables EF Core CLI tools to create DbContext at design time
- Used for migrations: `dotnet ef migrations add`, `dotnet ef database update`

**Connection String:**
- Hardcoded: `Server=(localdb)\\MSSQLLocalDB;Database=RetailMonolith;Trusted_Connection=True;MultipleActiveResultSets=true`

## Main Request Flows

### Flow 1: Browse Products and Add to Cart

```
User Request → /Products
  ↓
Pages/Products/IndexModel.OnGetAsync()
  ↓
AppDbContext.Products.Where(p => p.IsActive).ToListAsync()
  ↓
Render product list with "Add to Cart" forms
```

```
User Submits Form → POST /Products
  ↓
Pages/Products/IndexModel.OnPostAsync(productId)
  ↓
[ISSUE: Duplicate logic]
  ├─→ Direct DbContext manipulation (creates cart, adds line)
  └─→ CartService.AddToCartAsync() [redundant]
  ↓
Redirect → /Cart
```

### Flow 2: View Shopping Cart

```
User Request → GET /Cart
  ↓
Pages/Cart/IndexModel.OnGetAsync()
  ↓
CartService.GetCartWithLinesAsync("guest")
  ↓
AppDbContext.Carts.Include(c => c.Lines).FirstOrDefaultAsync()
  ↓
Project to tuples for display
  ↓
Render cart with total
```

### Flow 3: Checkout via API

```
API Request → POST /api/checkout
  ↓
CheckoutService.CheckoutAsync("guest", "tok_test")
  ↓
1. Load cart with lines
  ↓
2. Calculate total
  ↓
3. Reserve inventory (decrement quantities)
  ↓
4. Call PaymentGateway.ChargeAsync()
  ↓
5. Create Order (status = Paid/Failed based on payment)
  ↓
6. Clear cart lines
  ↓
7. SaveChanges() - commits all
  ↓
Return order summary {Id, Status, Total}
```

### Flow 4: Get Order Details via API

```
API Request → GET /api/orders/{id}
  ↓
AppDbContext.Orders.Include(o => o.Lines).SingleOrDefaultAsync(o => o.Id == id)
  ↓
Return order JSON or 404
```

## Areas of Coupling and Hotspots

### 1. **Tight Database Coupling**

**Issue**: All modules directly depend on `AppDbContext`
- Service layer directly uses EF Core
- No repository pattern or data access abstraction
- Difficult to unit test without database

**Impact**:
- Hard to swap database providers
- Testing requires integration tests or heavy mocking
- Migrations affect entire application

**Location**: All services and some page models

### 2. **Hardcoded "guest" Customer ID**

**Issue**: Customer identification is hardcoded throughout
- No authentication or session management
- Single shared cart across all users

**Impact**:
- Not production-ready for multi-user scenarios
- Refactoring to add auth requires changes in many places

**Locations**:
- `Pages/Products/IndexModel.OnPostAsync()`
- `Pages/Cart/IndexModel.OnGetAsync()`
- `CheckoutService.CheckoutAsync()`
- Minimal API endpoint `/api/checkout`

### 3. **Duplicate Cart Logic in Products Page**

**Issue**: `Pages/Products/IndexModel.OnPostAsync()` contains both:
- Direct DbContext cart manipulation
- Call to `CartService.AddToCartAsync()`

**Impact**:
- Potential for inconsistent state
- Code duplication and confusion
- Maintenance burden

**Location**: `Pages/Products/Index.cshtml.cs` lines 27-50

### 4. **Checkout Transaction Boundaries**

**Issue**: Inventory, payment, and order creation in single transaction
- Inventory decremented before payment
- Payment failure doesn't restore inventory
- No compensation logic

**Impact**:
- Inventory can be reserved but never paid for
- Failed payments leave stock unavailable
- Manual intervention required to fix

**Location**: `CheckoutService.CheckoutAsync()` method

### 5. **Lack of Inventory Validation in Cart**

**Issue**: `CartService.AddToCartAsync()` doesn't check inventory
- Users can add unlimited quantity to cart
- Validation only at checkout

**Impact**:
- Poor user experience (add to cart succeeds, checkout fails)
- Race conditions possible

**Location**: `CartService.AddToCartAsync()`

### 6. **Mock Payment Gateway in Production Path**

**Issue**: `MockPaymentGateway` always succeeds
- No error handling path tested
- Real payment integration not implemented

**Impact**:
- Application not production-ready
- Error handling untested

**Location**: `MockPaymentGateway.ChargeAsync()`

### 7. **Auto-Migration on Startup**

**Issue**: Migrations run automatically in `Program.cs`
- No control over migration timing
- Startup delay on first run
- Risk of migration failures blocking application start

**Impact**:
- Not suitable for production deployments
- Potential for schema changes during running deployments

**Location**: `Program.cs` lines 24-29

### 8. **No Concurrency Control**

**Issue**: Inventory updates lack pessimistic or optimistic locking
- Multiple simultaneous checkouts can cause race conditions

**Impact**:
- Inventory can go negative
- Overselling possible

**Location**: `CheckoutService.CheckoutAsync()` inventory reservation loop

### 9. **Denormalized Data Without Synchronization**

**Issue**: Cart and Order lines store product name/price snapshots
- No mechanism to update if product changes
- Intentional design, but not documented

**Impact**:
- Stale data in carts if products updated
- Acceptable for orders (immutable audit trail)

**Locations**: `CartLine`, `OrderLine` entities

## Technical Debt Observations

1. **Incomplete UI**: Checkout and Orders pages have no implementation
2. **No Authentication**: Hardcoded "guest" user throughout
3. **No Logging**: No structured logging or observability
4. **No Validation**: Minimal input validation on API endpoints
5. **No Error Handling**: Limited exception handling in services
6. **Test Coverage**: No unit tests observed
7. **Configuration**: Hardcoded values in DesignTimeDbContextFactory
8. **API Security**: No authentication/authorization on API endpoints
9. **CORS**: No CORS configuration (may be needed for frontend)
10. **Health Checks**: Registered but not customized (doesn't check DB health)
