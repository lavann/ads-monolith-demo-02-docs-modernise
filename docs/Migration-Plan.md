# Migration Plan - Monolith to Microservices

## Overview

This document outlines the step-by-step migration plan from the current monolithic Retail Application to a containerized microservices architecture using the **Strangler Fig Pattern**. The migration is designed to be incremental, safe, and reversible at each stage.

## Migration Strategy: Strangler Fig Pattern

The Strangler Fig pattern allows us to:
1. Build new functionality around the edges of the monolith
2. Gradually migrate functionality to new services
3. Retire monolith components incrementally
4. Maintain system functionality throughout the migration
5. Rollback individual services without affecting the entire system

### Key Principles
- **One service at a time**: Extract and stabilize before moving to the next
- **Feature flags**: Control routing between monolith and services
- **Parallel run**: Run old and new side-by-side during validation
- **Incremental database migration**: Start with shared database, move to service-owned databases
- **No big-bang**: Each phase is independently deployable and testable

## Migration Phases

### Phase 0: Foundation & Preparation
**Duration**: 2 weeks  
**Goal**: Establish infrastructure and tooling for containerized deployments

#### Tasks
1. **Containerize Existing Monolith**
   - Create `Dockerfile` for RetailMonolith
   - Multi-stage build (SDK for build, ASP.NET runtime for deployment)
   - Optimize image size (<200 MB)
   - Test local container deployment

2. **Set Up Container Orchestration**
   - Create `docker-compose.yml` for local development
   - Define network configuration
   - Set up SQL Server container
   - Configure volume mounts for data persistence

3. **Implement Health Checks**
   - Add `/health` endpoint to monolith
   - Include database connectivity check
   - Configure health check in Program.cs
   - Test with container health checks

4. **Establish Observability**
   - Add structured logging (Serilog or Microsoft.Extensions.Logging)
   - Configure correlation IDs for request tracking
   - Set up local logging dashboard (optional: ELK stack)
   - Add basic metrics endpoints

5. **Set Up CI/CD Pipeline**
   - GitHub Actions or Azure DevOps pipeline
   - Automated build and test
   - Container image build and push to registry
   - Deploy to development environment

6. **Create Test Harness**
   - Integration test suite for critical paths
   - Smoke tests for container deployment
   - Performance baseline metrics
   - Database seeding for test environments

#### Deliverables
- ✅ Containerized monolith running in Docker
- ✅ docker-compose.yml for local multi-container setup
- ✅ Health check endpoints implemented
- ✅ CI/CD pipeline building and deploying containers
- ✅ Test suite with >80% coverage of critical paths
- ✅ Runbook for container operations

#### Success Criteria
- Monolith runs in container with no functionality loss
- All existing tests pass in containerized environment
- Health checks report healthy status
- CI/CD pipeline deploys to dev environment successfully

#### Rollback Plan
- Revert to non-containerized deployment
- No code changes to monolith, only infrastructure additions
- Low risk: containerization should be transparent to application logic

---

### Phase 1: Extract Payment Service (FIRST SLICE)
**Duration**: 2-3 weeks  
**Goal**: Extract the Payment Gateway into an independent service

#### Why Payment Service First?
This is the **optimal first slice** for several reasons:

1. **Well-Defined Interface**: `IPaymentGateway` is already abstracted
   ```csharp
   Task<PaymentResult> ChargeAsync(PaymentRequest req, CancellationToken ct)
   ```

2. **No Database Dependencies**: Payment service is stateless
   - No data migration required
   - No schema changes
   - Simplest possible extraction

3. **Easily Reversible**: Can switch between implementations via DI configuration
   ```csharp
   // Option 1: Monolith
   services.AddScoped<IPaymentGateway, MockPaymentGateway>();
   
   // Option 2: Service
   services.AddScoped<IPaymentGateway, HttpPaymentGateway>();
   ```

4. **Low Risk**: Payment is only called during checkout
   - Limited blast radius
   - Easy to test end-to-end
   - Not on critical path for browsing/cart operations

5. **High Value**: Demonstrates end-to-end service extraction
   - Containerization
   - Service-to-service communication
   - Feature flag implementation
   - Monitoring and observability

6. **Demoable**: Clear before/after demonstration
   - Call to in-process mock vs HTTP service
   - Observable network traffic
   - Measurable latency impact

#### Tasks

**1. Create Payment Service Project**
- New ASP.NET Core Web API project: `PaymentService`
- Implement `POST /api/payments/charge` endpoint
- Use same `PaymentRequest` and `PaymentResult` models
- Currently: Mock implementation (real gateway integration later)
- Add health check endpoint
- Add OpenAPI/Swagger documentation

**2. Containerize Payment Service**
- Create `Dockerfile` for PaymentService
- Add to `docker-compose.yml`
- Configure environment variables for payment provider
- Test independent startup

**3. Implement HTTP Client in Monolith**
- Create `HttpPaymentGateway : IPaymentGateway`
- Use `HttpClient` with Polly for resilience:
  - Retry policy (3 attempts, exponential backoff)
  - Circuit breaker (open after 5 failures in 30s)
  - Timeout (5 seconds)
- Add configuration for service URL
- Add telemetry and logging

**4. Add Feature Flag**
- Use configuration-based feature flag
  ```json
  {
    "FeatureFlags": {
      "UsePaymentService": false
    }
  }
  ```
- Factory pattern to select implementation:
  ```csharp
  if (config["FeatureFlags:UsePaymentService"] == "true")
      services.AddScoped<IPaymentGateway, HttpPaymentGateway>();
  else
      services.AddScoped<IPaymentGateway, MockPaymentGateway>();
  ```

**5. Deploy and Test**
- Deploy both containers (monolith + payment service)
- Run checkout flow with feature flag OFF (baseline)
- Run checkout flow with feature flag ON (new service)
- Compare results and latency
- Smoke test: 100 checkouts with service enabled

**6. Gradual Rollout**
- Enable feature flag in dev environment
- Run for 1 week, monitor for errors
- Enable in staging environment
- Run for 1 week with production-like load
- Enable in production (start at 10%, ramp to 100%)

#### Deliverables
- ✅ PaymentService container running independently
- ✅ HttpPaymentGateway implementation in monolith
- ✅ Feature flag controlling routing
- ✅ Integration tests for both paths
- ✅ Monitoring dashboard for payment service

#### Success Criteria
- Payment service handles 100% of payments without errors
- P95 latency < 200ms for payment calls
- No increase in payment failures
- Logs show correlation IDs across services
- Rollback to mock completes in < 5 minutes

#### Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Network latency increases checkout time | Medium | Set aggressive timeout (5s), use circuit breaker |
| Payment service unavailable | High | Circuit breaker falls back to mock, alerting on fallback |
| Lost payment requests | High | Implement idempotency with request IDs, retry logic |
| Increased infrastructure cost | Low | Single small container, minimal resource usage |

#### Rollback Plan
1. Set feature flag `UsePaymentService: false`
2. Restart monolith (picks up config change)
3. Payment processing reverts to in-process mock
4. Total rollback time: < 5 minutes
5. No data loss (no database involved)

---

### Phase 2: Extract Products Service
**Duration**: 3-4 weeks  
**Goal**: Separate product catalog into independent service with database

#### Tasks

**1. Create Products Service Project**
- New ASP.NET Core Web API project: `ProductsService`
- Implement REST endpoints:
  - `GET /api/products` - List active products
  - `GET /api/products/{id}` - Get product by ID
  - `GET /api/products/sku/{sku}` - Get product by SKU
  - `POST /api/products` - Create product (admin)
  - `PUT /api/products/{id}` - Update product (admin)
  - `DELETE /api/products/{id}` - Soft delete product (admin)

**2. Database Strategy: Shared Schema Initially**
- Products service accesses `Products` table via database view
- Create view in shared database: `vw_ProductsService`
- Grants service read/write access via view
- Schema ownership remains in monolith initially
- Prepare for future data migration

**3. Update Monolith to Call Products Service**
- Create `IProductsRepository` interface
- Implement `HttpProductsRepository` (calls service via HTTP)
- Implement `DbProductsRepository` (current direct EF access)
- Feature flag: `UseProductsService`
- Update `Pages/Products/IndexModel` to use repository

**4. Caching Layer**
- Add Redis cache in front of Products service
- Cache product listings (5-minute TTL)
- Cache individual products (10-minute TTL)
- Invalidate cache on product updates

**5. Migrate Database Schema (Later Substep)**
- Once stable, migrate `Products` table to Products service database
- Use database sync tool during transition
- Update connection strings to point to service DB
- Remove view and direct access from monolith

#### Deliverables
- ✅ Products service with REST API
- ✅ Repository pattern in monolith
- ✅ Redis caching for product data
- ✅ Performance benchmarks (target: <50ms P95)
- ✅ Database migration scripts (if schema moved)

#### Success Criteria
- All product browsing operations work via service
- Cache hit rate > 80% for product listings
- No increase in page load time
- Products service scales to 2+ instances
- Database query count reduced by 80% (due to caching)

#### Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Products service slow | High | Implement caching, set aggressive timeout |
| Database contention | Medium | Use read replicas, optimize indexes |
| Cache inconsistency | Medium | Short TTL, cache invalidation on writes |
| API versioning issues | Medium | Implement versioned API endpoints |

#### Rollback Plan
1. Set feature flag `UseProductsService: false`
2. Monolith reverts to direct database access
3. Products service continues running (no impact)
4. Total rollback time: < 5 minutes

---

### Phase 3: Extract Inventory Service
**Duration**: 3-4 weeks  
**Goal**: Separate inventory management with event-based updates

#### Tasks

**1. Create Inventory Service**
- New ASP.NET Core Web API project: `InventoryService`
- Implement REST endpoints:
  - `GET /api/inventory/{sku}` - Get stock level
  - `POST /api/inventory/reserve` - Reserve inventory (checkout)
  - `POST /api/inventory/release` - Release reservation (cancel)
  - `PUT /api/inventory/{sku}` - Update stock level (admin)

**2. Implement Event-Based Updates**
- Add message broker (RabbitMQ or Azure Service Bus)
- Publish events:
  - `InventoryReserved` (sku, quantity, orderId)
  - `InventoryInsufficient` (sku, requested, available)
  - `InventoryUpdated` (sku, newQuantity)
- Subscribe to events:
  - `OrderCreated` → Reserve inventory
  - `OrderCancelled` → Release inventory

**3. Implement Pessimistic Locking**
- Use database row-level locks during reservation
- Prevent overselling under concurrent checkouts
- Add timeout for lock acquisition (5 seconds)
- Return error if lock cannot be acquired

**4. Update Checkout Flow**
- CheckoutService calls Inventory service via HTTP
- Handle `InventoryInsufficient` errors gracefully
- Add compensation logic: release inventory on payment failure
- Implement saga pattern for distributed transaction

**5. Monitoring & Alerts**
- Track inventory reservation success rate
- Alert on low stock levels
- Alert on high reservation failure rate
- Dashboard for inventory metrics

#### Deliverables
- ✅ Inventory service with REST API
- ✅ Event-based communication (message broker)
- ✅ Pessimistic locking for concurrency control
- ✅ Saga pattern for checkout orchestration
- ✅ Monitoring and alerting

#### Success Criteria
- No inventory overselling (stock never goes negative)
- Reservation success rate > 95%
- Inventory reservation latency P95 < 100ms
- Failed payments release inventory within 10 seconds
- Events processed with < 1-second latency

#### Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Inventory service unavailable | High | Circuit breaker, fallback to fail-safe (reject order) |
| Event message loss | High | Use durable queues, implement at-least-once delivery |
| Distributed transaction complexity | Medium | Implement saga with compensation logic |
| Database deadlocks | Medium | Optimize lock acquisition order, add timeouts |

#### Rollback Plan
1. Set feature flag `UseInventoryService: false`
2. CheckoutService reverts to direct database access
3. Inventory service continues processing events (no harm)
4. Total rollback time: < 5 minutes

---

### Phase 4: Extract Cart Service
**Duration**: 3-4 weeks  
**Goal**: Externalize shopping cart with session management

#### Tasks

**1. Create Cart Service**
- New ASP.NET Core Web API project: `CartService`
- Implement REST endpoints:
  - `GET /api/cart/{customerId}` - Get cart
  - `POST /api/cart/{customerId}/items` - Add item
  - `PUT /api/cart/{customerId}/items/{sku}` - Update quantity
  - `DELETE /api/cart/{customerId}/items/{sku}` - Remove item
  - `DELETE /api/cart/{customerId}` - Clear cart

**2. Use Redis for Cart Storage**
- Store carts in Redis instead of SQL database
- TTL: 7 days for inactive carts
- JSON serialization for cart objects
- Key format: `cart:{customerId}`

**3. Implement Cart Enrichment**
- Cart service stores only SKU and quantity
- On retrieval, call Products service for current price/name
- Return enriched cart with product details
- Handle products that no longer exist

**4. Session Management**
- Integrate with authentication system (future: JWT tokens)
- Currently: Continue using "guest" customer ID
- Prepare for multi-user scenario

**5. Update Monolith**
- Remove `ICartService` implementation from monolith
- Create `HttpCartService` implementation
- Update Razor Pages to call Cart service
- Feature flag: `UseCartService`

#### Deliverables
- ✅ Cart service with REST API
- ✅ Redis for cart storage
- ✅ Cart enrichment with product data
- ✅ Updated monolith to use Cart service
- ✅ Session management (preparation for auth)

#### Success Criteria
- Cart operations complete in < 100ms P95
- Cart data persists across browser sessions
- Old carts expire after 7 days
- No cart data loss during service restart
- Cart service scales horizontally (stateless)

#### Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Redis unavailable | High | Implement fallback to database or in-memory cache |
| Stale product prices in cart | Medium | Accept as designed: prices snapshot at add-to-cart time |
| Cart data loss | Low | Redis persistence (AOF), regular snapshots |

#### Rollback Plan
1. Set feature flag `UseCartService: false`
2. Monolith reverts to database-backed cart
3. Carts in Redis remain (no harm, will expire)
4. Total rollback time: < 5 minutes

---

### Phase 5: Extract Orders Service
**Duration**: 4-5 weeks  
**Goal**: Separate order management with saga orchestration

#### Tasks

**1. Create Orders Service**
- New ASP.NET Core Web API project: `OrdersService`
- Implement REST endpoints:
  - `GET /api/orders` - List orders for customer
  - `GET /api/orders/{id}` - Get order details
  - `POST /api/orders/checkout` - Process checkout (saga orchestrator)
  - `PUT /api/orders/{id}/status` - Update order status
  - `PUT /api/orders/{id}/ship` - Ship order

**2. Implement Saga Pattern for Checkout**
- Orders service orchestrates checkout saga:
  1. Get cart from Cart service
  2. Reserve inventory via Inventory service
  3. Charge payment via Payment service
  4. Create order in Orders database
  5. Clear cart via Cart service
- Handle failures with compensation:
  - Payment fails → Release inventory
  - Order creation fails → Refund payment, release inventory

**3. Event Publishing**
- Publish events:
  - `OrderCreated` (orderId, customerId, total, lineItems)
  - `OrderPaid` (orderId, paymentRef)
  - `OrderFailed` (orderId, reason)
  - `OrderShipped` (orderId, trackingNumber)

**4. Update Monolith**
- Remove checkout logic from monolith
- Proxy `/api/checkout` to Orders service
- Feature flag: `UseOrdersService`
- Update Razor Pages to call Orders service

**5. Database Migration**
- Migrate `Orders` and `OrderLines` tables to Orders service database
- Use event sourcing for order history (optional enhancement)
- Remove direct access from monolith

#### Deliverables
- ✅ Orders service with saga orchestration
- ✅ Event publishing for order lifecycle
- ✅ Compensation logic for failures
- ✅ Database migration to Orders service
- ✅ Updated monolith

#### Success Criteria
- Checkout success rate unchanged (>98%)
- Order creation latency P95 < 500ms
- 100% of failed payments result in inventory release
- Orders database independent of monolith
- Saga handles all failure scenarios gracefully

#### Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Saga complexity | High | Thorough testing of all failure paths, add logging |
| Partial failures leave system inconsistent | High | Implement idempotent operations, compensation logic |
| Long-running saga timeouts | Medium | Set aggressive timeouts, fail fast |
| Database migration downtime | Medium | Use blue-green deployment, sync data during transition |

#### Rollback Plan
1. Set feature flag `UseOrdersService: false`
2. Monolith resumes checkout orchestration
3. Orders service continues receiving new orders
4. Total rollback time: < 10 minutes

---

### Phase 6: Retire Monolith Frontend Components
**Duration**: 2-3 weeks  
**Goal**: Transition UI to dedicated frontend or complete backend decomposition

#### Tasks

**1. Convert Monolith to API Gateway**
- Remove business logic from Razor Pages
- Keep Razor Pages as thin UI layer
- Proxy all API calls to backend services
- Implement backend-for-frontend (BFF) pattern

**2. (Optional) Replace with SPA Frontend**
- Build React/Vue/Angular SPA
- Call backend services via API Gateway
- Implement client-side routing
- Use JWT authentication

**3. Implement API Gateway**
- Deploy YARP (Yet Another Reverse Proxy) or Ocelot
- Configure routing rules
- Add authentication middleware
- Add rate limiting and throttling

**4. Complete Database Migration**
- Ensure all services have independent databases
- Remove shared database connections
- Validate no cross-service database access

#### Deliverables
- ✅ API Gateway with routing rules
- ✅ Monolith reduced to thin UI layer or retired
- ✅ (Optional) SPA frontend
- ✅ All services with independent databases

#### Success Criteria
- No direct database access from monolith
- All business logic in services
- API Gateway handles 100% of traffic routing
- Frontend fully decoupled from backend services

---

## Timeline Summary

| Phase | Duration | Milestone |
|-------|----------|-----------|
| Phase 0: Foundation | 2 weeks | Containerized monolith |
| Phase 1: Payment Service | 2-3 weeks | First service extracted |
| Phase 2: Products Service | 3-4 weeks | First database-backed service |
| Phase 3: Inventory Service | 3-4 weeks | Event-driven architecture |
| Phase 4: Cart Service | 3-4 weeks | Redis-backed service |
| Phase 5: Orders Service | 4-5 weeks | Saga orchestration |
| Phase 6: Retire Monolith | 2-3 weeks | Full decomposition |
| **Total** | **19-26 weeks** | **~5-6 months** |

## First Slice Justification: Payment Service

### Why Payment Service is the Ideal Starting Point

**Technical Simplicity**:
- ✅ No database dependencies (stateless)
- ✅ Well-defined interface (already abstracted)
- ✅ Single responsibility (process payments)
- ✅ No complex business logic
- ✅ Independent of other domains

**Low Risk**:
- ✅ Easy to rollback (feature flag toggle)
- ✅ Limited blast radius (only affects checkout)
- ✅ No data migration required
- ✅ No schema changes
- ✅ Can run side-by-side with mock for validation

**High Learning Value**:
- ✅ Demonstrates full extraction lifecycle
- ✅ Establishes patterns for future services
- ✅ Validates container orchestration
- ✅ Tests service-to-service communication
- ✅ Proves feature flag mechanism

**Demoable**:
- ✅ Clear before/after comparison
- ✅ Observable network traffic (monolith → service)
- ✅ Measurable latency impact
- ✅ Easy to explain to stakeholders

**Business Value**:
- ✅ Isolates payment processing (PCI compliance preparation)
- ✅ Enables independent scaling of payment logic
- ✅ Prepares for real payment gateway integration
- ✅ Reduces monolith complexity incrementally

### Alternatives Considered

**Products Service First**:
- ❌ Requires database migration or shared database complexity
- ❌ High read volume = performance risk
- ❌ More complex caching strategy needed
- ✅ High business value (often accessed)
- **Verdict**: Good second choice, but more complex than Payment

**Cart Service First**:
- ❌ Requires Redis setup (new infrastructure)
- ❌ Session management complexity
- ❌ Affects user experience directly (high risk)
- ✅ Good isolation from other domains
- **Verdict**: Better suited for Phase 4 after patterns established

**Inventory Service First**:
- ❌ Requires event infrastructure (message broker)
- ❌ Concurrency control complexity (locking)
- ❌ Critical for checkout (high risk if bugs)
- ✅ Good isolation from other domains
- **Verdict**: Too complex for first extraction

## Success Metrics

### Technical Metrics
- **Service Availability**: >99.9% uptime per service
- **API Latency**: P95 < 200ms for all endpoints
- **Error Rate**: <0.1% for all service calls
- **Deployment Frequency**: Daily deployments without downtime
- **Rollback Time**: <5 minutes for any service

### Business Metrics
- **Checkout Success Rate**: Maintained at >98%
- **Page Load Time**: No degradation (maintain <2s)
- **Cart Abandonment**: No increase
- **Customer Complaints**: No increase related to performance

### Operational Metrics
- **Mean Time to Recovery (MTTR)**: <30 minutes
- **Change Failure Rate**: <5%
- **Lead Time for Changes**: <1 day (from commit to production)
- **Deployment Success Rate**: >95%

## Risk Management

### Overall Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Service communication failures | Medium | High | Circuit breakers, retries, fallbacks |
| Database migration data loss | Low | Critical | Blue-green deployment, data sync, backups |
| Performance degradation | Medium | Medium | Load testing, caching, monitoring |
| Increased operational complexity | High | Medium | Automation, runbooks, training |
| Cost overrun | Low | Low | Start small, scale based on need |
| Team skill gaps | Medium | Medium | Training, pair programming, documentation |

### Rollback Strategy (Global)
Every phase has:
1. **Feature flags**: Instant routing control
2. **Parallel run**: Old and new run simultaneously during validation
3. **Automated tests**: Catch regressions before production
4. **Monitoring**: Detect issues quickly
5. **Rollback playbook**: Step-by-step rollback instructions

## Conclusion

This migration plan provides a safe, incremental path from monolith to microservices. The **Payment Service** is selected as the first slice due to its simplicity, low risk, and high learning value. Each subsequent phase builds on the patterns and infrastructure established in previous phases, gradually decomposing the monolith while maintaining system stability and behavior.

The plan is designed to be **achievable with the current stack** (ASP.NET Core, EF Core, SQL Server, Docker) and requires **no big-bang rewrite**. Each phase is independently testable, deployable, and reversible, ensuring business continuity throughout the migration.

**Next Steps**: Obtain stakeholder approval for Phase 1 (Payment Service extraction) before beginning implementation.
