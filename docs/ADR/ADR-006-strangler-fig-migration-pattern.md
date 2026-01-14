# ADR-006: Strangler Fig Migration Pattern

## Status
Accepted

## Context

The Retail Monolith application needs to be migrated from a monolithic architecture to microservices. We must choose a migration strategy that:

1. Minimizes risk to the running production system
2. Allows incremental migration over several months
3. Enables rollback at any stage
4. Maintains business continuity throughout migration
5. Avoids a "big-bang" rewrite
6. Allows teams to learn and adapt during migration

Common migration approaches include:
- **Big-Bang Rewrite**: Build entire new system, then switch over
- **Branch by Abstraction**: Create abstractions, swap implementations behind them
- **Strangler Fig Pattern**: Gradually replace old system with new services
- **Parallel Run**: Run old and new systems simultaneously, gradually migrate users

## Decision

Adopt the **Strangler Fig Pattern** for migrating from monolith to microservices.

## What is Strangler Fig Pattern?

Named after the strangler fig tree that grows around a host tree, gradually replacing it, this pattern involves:

1. **Intercept**: Route requests through a facade/proxy
2. **Strangle**: Gradually implement new functionality in separate services
3. **Replace**: Direct traffic from monolith to new services incrementally
4. **Retire**: Remove old monolith code once fully replaced

```
Stage 1: Monolith Only          Stage 2: Mixed (Strangling)      Stage 3: Services Only
┌─────────────┐                 ┌─────────────┐                  ┌─────────────┐
│   Client    │                 │   Client    │                  │   Client    │
└──────┬──────┘                 └──────┬──────┘                  └──────┬──────┘
       │                               │                                 │
       ▼                               ▼                                 ▼
┌─────────────┐                 ┌─────────────┐                  ┌─────────────┐
│  Monolith   │                 │   Facade/   │                  │ API Gateway │
│             │                 │    Router   │                  └──────┬──────┘
│ - Products  │                 └──────┬──────┘                         │
│ - Cart      │                    ┌───┴───┐                     ┌──────┴──────┐
│ - Checkout  │                    │       │                     │             │
│ - Orders    │              ┌─────▼──┐  ┌─▼────────┐      ┌─────▼──┐   ┌────▼────┐
│ - Payment   │              │Monolith│  │ Payment  │      │Products│   │ Payment │
└─────────────┘              │(partial)│ │ Service  │      │Service │   │ Service │
                             │         │  └──────────┘      └────────┘   └─────────┘
                             │-Products│                    (More services...)
                             │- Cart   │
                             │-Checkout│
                             │- Orders │
                             └─────────┘
```

## Rationale

### Why Strangler Fig Pattern

1. **Incremental Risk**
   - Extract one service at a time
   - Validate before moving to next service
   - Small changes reduce probability of major failures
   - Can pause or adjust based on learnings

2. **Continuous Delivery**
   - System remains operational during entire migration
   - No "migration downtime"
   - Features can be delivered while migration is in progress
   - Business value continues to flow

3. **Easy Rollback**
   - Each service extraction is independently reversible
   - Feature flags enable instant routing changes
   - Can revert to monolith with configuration change
   - No data loss or major rollback operations

4. **Learning and Adaptation**
   - First extractions teach patterns for later ones
   - Team builds expertise incrementally
   - Architecture can be adjusted based on early learnings
   - Mistakes have limited impact

5. **Parallel Operation**
   - Old and new run side-by-side during transition
   - Can validate new service against monolith
   - Gradual traffic migration (1% → 10% → 50% → 100%)
   - A/B testing possible

### Key Enablers

#### 1. Feature Flags
Control which implementation handles requests:

```csharp
// Example: Payment Gateway Selection
public void ConfigureServices(IServiceCollection services)
{
    var usePaymentService = Configuration.GetValue<bool>("FeatureFlags:UsePaymentService");
    
    if (usePaymentService)
    {
        // Route to new Payment Service via HTTP
        services.AddScoped<IPaymentGateway, HttpPaymentGateway>();
    }
    else
    {
        // Use in-process implementation (monolith)
        services.AddScoped<IPaymentGateway, MockPaymentGateway>();
    }
}
```

**Benefits**:
- Instant rollback (change config, restart)
- Gradual rollout (enable for subset of requests)
- A/B testing (compare old vs new)
- No code changes needed to switch

#### 2. API Gateway / Facade Pattern
Central routing point for all requests:

```
┌─────────────────────────────────────────────┐
│           API Gateway / Facade               │
│                                              │
│  Routing Rules:                             │
│  - /api/payments/*  → Payment Service       │
│  - /api/products/*  → Products Service      │
│  - /api/cart/*      → Cart Service          │
│  - /* (fallback)    → Monolith              │
└─────────────────────────────────────────────┘
```

**Implementation Options**:
- **YARP** (Yet Another Reverse Proxy) - .NET-based
- **Ocelot** - .NET API Gateway
- **Nginx** - Traditional reverse proxy
- **Azure API Management** - Managed service

#### 3. Abstraction Layers
Interface-based design enables swapping implementations:

```csharp
// Abstraction
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(PaymentRequest req);
}

// Old Implementation (in monolith)
public class MockPaymentGateway : IPaymentGateway { ... }

// New Implementation (calls service)
public class HttpPaymentGateway : IPaymentGateway
{
    private readonly HttpClient _http;
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest req)
    {
        var response = await _http.PostAsJsonAsync("/api/payments/charge", req);
        return await response.Content.ReadFromJsonAsync<PaymentResult>();
    }
}
```

**Benefits**:
- No changes to calling code
- Clean separation of concerns
- Testable independently
- Enables feature flag switching

#### 4. Shared Database (Transitional)
Initially, services can share database:

```
┌──────────────┐
│   Monolith   │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────┐
│   Database   │◀────│  Products   │
│              │     │  Service    │
│  (Shared)    │     └─────────────┘
│              │
│ - Products   │     ┌─────────────┐
│ - Inventory  │◀────│  Inventory  │
│ - Cart       │     │  Service    │
│ - Orders     │     └─────────────┘
└──────────────┘
```

**Later, split databases**:

```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│   Monolith   │     │  Products   │◀────│  Products DB │
└──────┬───────┘     │  Service    │     └──────────────┘
       │             └─────────────┘
       ▼                    ▲
┌──────────────┐            │ (Events)
│  Orders DB   │            │
└──────────────┘     ┌─────────────┐     ┌──────────────┐
                     │  Inventory  │◀────│ Inventory DB │
                     │  Service    │     └──────────────┘
                     └─────────────┘
```

#### 5. Event-Driven Communication
Decouple services via asynchronous events:

```csharp
// Service publishes event
await _eventBus.PublishAsync(new OrderCreatedEvent
{
    OrderId = order.Id,
    CustomerId = order.CustomerId,
    Total = order.Total
});

// Other services subscribe
public class InventoryService
{
    public async Task HandleOrderCreated(OrderCreatedEvent evt)
    {
        // Reserve inventory for order
        await ReserveInventoryAsync(evt.OrderId);
    }
}
```

**Benefits**:
- Services don't need to know about each other
- Temporal decoupling (receiver can be offline temporarily)
- Easy to add new subscribers
- Replay events for new services

## Implementation Strategy

### Phase-by-Phase Extraction

#### Phase 0: Preparation (Foundation)
**Goal**: Set up infrastructure for strangler pattern

**Tasks**:
1. Containerize monolith
2. Implement health check endpoints
3. Set up feature flag infrastructure
4. Create abstraction interfaces for services
5. Establish CI/CD pipelines
6. Set up monitoring and logging

**Outcome**: Monolith ready for gradual extraction

---

#### Phase 1: First Service (Payment)
**Goal**: Extract simplest service to validate pattern

**Selection Criteria for First Service**:
- Minimal dependencies (ideally stateless)
- Well-defined interface (already abstracted)
- Low risk (limited blast radius)
- Easy to validate (clear success/failure criteria)

**Note**: See ADR-007 for detailed justification of Payment Service selection.

**Tasks**:
1. Create Payment Service project
2. Implement `POST /api/payments/charge` endpoint
3. Create `HttpPaymentGateway` in monolith
4. Add feature flag: `UsePaymentService`
5. Deploy both containers side-by-side
6. Validate with feature flag OFF (baseline)
7. Enable feature flag (gradually: 1% → 10% → 100%)
8. Monitor for errors and latency
9. Remove old implementation once 100% traffic on service

**Rollback Plan**: Set feature flag to `false`, restart monolith

**Success Criteria**: 100% of payments processed by service with no increase in errors

---

#### Phase 2-5: Subsequent Services
Apply same pattern to each domain:
- **Phase 2**: Products Service (introduces database complexity)
- **Phase 3**: Inventory Service (introduces event-driven architecture)
- **Phase 4**: Cart Service (introduces Redis/caching)
- **Phase 5**: Orders Service (introduces saga pattern)

Each phase follows same pattern:
1. Extract service
2. Create abstraction in monolith
3. Feature flag for routing
4. Gradual rollout
5. Validate and monitor
6. Full cutover
7. Retire monolith code

---

#### Phase 6: Retirement
**Goal**: Remove monolith entirely (optional)

**Tasks**:
1. Convert monolith to thin API Gateway (if keeping UI)
2. Remove all business logic from monolith
3. (Optional) Replace with SPA frontend
4. Shut down monolith container

**Outcome**: Pure microservices architecture

## Consequences

### Positive

1. **Risk Mitigation**
   - Small, incremental changes
   - Easy to rollback individual services
   - System remains stable throughout migration
   - Team learns and adapts during process

2. **Business Continuity**
   - No migration downtime
   - Features continue to be delivered
   - Revenue not impacted
   - Users experience no disruption

3. **Flexibility**
   - Can pause migration if needed
   - Can adjust approach based on learnings
   - Architecture decisions validated early
   - Can prioritize high-value services first

4. **Team Development**
   - Team builds expertise incrementally
   - Less overwhelming than big-bang
   - Expertise shared across team over time
   - Documentation and patterns emerge naturally

5. **Technical Validation**
   - Patterns proven before widespread adoption
   - Infrastructure validated with single service
   - Performance characteristics understood early
   - Can adjust architecture if needed

### Negative

1. **Extended Timeline**
   - Migration takes months instead of weeks
   - Prolonged period of mixed architecture
   - Technical debt exists longer
   - Team context-switches between old and new

2. **Dual Maintenance**
   - Must maintain both monolith and services
   - Bug fixes may need to be applied in both places
   - Testing complexity increases temporarily
   - Operational overhead higher during transition

3. **Transitional Complexity**
   - Feature flags add conditional logic
   - Routing layer adds latency
   - Debugging across monolith and services harder
   - Monitoring must cover both old and new

4. **Incomplete Benefits**
   - Microservices benefits not fully realized until complete
   - Can't fully retire monolith database until all services extracted
   - Some operational complexity without full scalability benefits

5. **Requires Discipline**
   - Team must resist temptation to "just do big-bang"
   - Must stick to incremental approach
   - Requires good communication and planning
   - Feature flags must be cleaned up eventually

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Feature flag proliferation | Medium | Document all flags, plan removal, use feature flag management tool |
| Services run old monolith code accidentally | High | Clear naming, automated tests, code reviews |
| Database schema conflicts during transition | High | Coordinate schema changes, use database views for abstraction |
| Performance degradation from network calls | Medium | Use caching, measure latency, set timeouts |
| Team loses momentum | Medium | Celebrate milestones, show progress, maintain roadmap |

## Alternatives Considered

### Big-Bang Rewrite
**Approach**: Build entire new microservices architecture, then switch over

**Pros**:
- Clean slate, no technical debt
- Fastest to "pure" microservices
- No dual maintenance

**Cons**:
- ❌ High risk (all or nothing)
- ❌ Long time to business value
- ❌ Hard to rollback
- ❌ Business features on hold during rewrite
- ❌ Difficult to estimate accurately

**Verdict**: Too risky for production system with active users

### Branch by Abstraction
**Approach**: Create abstractions in monolith, gradually swap implementations, never split deployment

**Pros**:
- Gradual refactoring
- No network calls
- Single deployment

**Cons**:
- ❌ Doesn't achieve true microservices (still single deployment)
- ❌ Monolith grows in complexity
- ❌ Cannot scale services independently
- ❌ Fails to meet containerization requirement

**Verdict**: Good for refactoring, but doesn't achieve target architecture

### Parallel Run (Complete Rewrite + Shadow Mode)
**Approach**: Build complete new system, run in parallel with traffic duplication, then cutover

**Pros**:
- Can validate new system thoroughly
- Clean architecture from start

**Cons**:
- ❌ Requires 2x infrastructure during transition
- ❌ Complex traffic duplication logic
- ❌ Data consistency challenges
- ❌ Still big-bang cutover at end

**Verdict**: Too complex and expensive for this project

### Blue-Green Deployment (Service Level)
**Approach**: Deploy new version alongside old, switch all traffic at once

**Pros**:
- Fast cutover
- Easy rollback

**Cons**:
- ❌ Still all-or-nothing per service
- ❌ No gradual rollout
- ❌ Higher risk than incremental approach

**Verdict**: Good for individual service deployments, but Strangler is better for overall migration strategy

## Success Metrics

### Technical Metrics
- **Service Extraction Time**: Each service < 4 weeks
- **Rollback Success Rate**: 100% (all rollbacks successful)
- **Downtime During Migration**: 0 hours
- **Service Availability**: >99.9% per service
- **Feature Flag Cleanup**: All flags removed within 2 weeks of 100% cutover

### Team Metrics
- **Team Confidence**: Survey shows increasing confidence over time
- **Knowledge Sharing**: All team members contribute to at least 2 service extractions
- **Documentation**: Runbooks exist for each service
- **Incident Response Time**: MTTR < 30 minutes

### Business Metrics
- **Feature Delivery Velocity**: Maintained or improved during migration
- **Customer Complaints**: No increase related to migration
- **Revenue Impact**: Zero negative impact from migration

## Related Decisions

- **ADR-001**: Monolithic Architecture (original decision, now evolving)
- **ADR-005**: Container-Based Deployment (enables strangler pattern)
- **ADR-007**: Payment Service as First Extraction (first service to strangle)

## References

- [Martin Fowler: StranglerFigApplication](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Sam Newman: Monolith to Microservices](https://samnewman.io/books/monolith-to-microservices/)
- [Microsoft: Strangler Fig Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig)
- [Migration Strategies for Microservices](https://www.nginx.com/blog/refactoring-a-monolith-into-microservices/)

## Conclusion

The Strangler Fig Pattern is the most appropriate migration strategy for this project because it:
- ✅ Minimizes risk through incremental changes
- ✅ Maintains business continuity throughout migration
- ✅ Enables learning and adaptation
- ✅ Provides easy rollback at each stage
- ✅ Delivers value continuously

While it takes longer than a big-bang rewrite, the reduced risk and maintained business continuity make it the superior choice for migrating a production system with active users.
