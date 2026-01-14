# ADR-007: Payment Service as First Service Extraction

## Status
Accepted

## Context

As part of the migration from monolith to microservices using the Strangler Fig Pattern (ADR-006), we need to select the first service to extract. This decision is critical because:

1. It establishes patterns and practices for subsequent extractions
2. It validates our infrastructure and tooling choices
3. It provides learning opportunities with manageable risk
4. It demonstrates feasibility to stakeholders
5. It sets the standard for success metrics

The monolith has four logical domains that could be extracted:
- **Payment**: Payment processing via external gateway
- **Products**: Product catalog management
- **Inventory**: Stock level tracking
- **Cart**: Shopping cart management
- **Orders**: Order processing and orchestration

## Decision

Extract the **Payment Service** as the first service.

The Payment Service will:
- Expose a REST API: `POST /api/payments/charge`
- Accept `PaymentRequest` (amount, currency, token)
- Return `PaymentResult` (succeeded, providerRef, error)
- Be called by the monolith's `CheckoutService` via HTTP
- Run as an independent container
- Be deployed alongside the monolith

## Rationale

### Why Payment Service?

#### 1. Technical Simplicity
**No Database Dependencies**
- Payment service is stateless
- No data migration required
- No schema changes or synchronization
- Eliminates database-related complexities for first extraction

**Well-Defined Interface**
- Already abstracted behind `IPaymentGateway` interface
- Single method: `ChargeAsync(PaymentRequest, CancellationToken)`
- Clear inputs and outputs
- No hidden dependencies

**Minimal Business Logic**
- Simple pass-through to payment provider
- No complex domain rules
- Easy to understand and implement
- Quick to develop and test

**Small Codebase**
- Minimal implementation (simple pass-through to gateway)
- Easy to replicate in new service
- Low risk of bugs during extraction
- Fast to build and deploy

#### 2. Low Risk Profile

**Limited Blast Radius**
- Only affects checkout flow
- Does not impact product browsing, cart management
- Users only encounter if they check out
- Typical e-commerce: <5% of visitors checkout

**Easy to Validate**
- Payment succeeds or fails (binary outcome)
- Easy to verify correctness
- Clear error conditions
- Simple integration testing

**Non-Critical Read Path**
- Not on hot path for browsing or cart operations
- Does not affect page load times for most users
- Performance impact limited to checkout

**Easily Reversible**
- Feature flag enables instant rollback
- No database changes to undo
- No data consistency issues
- Can revert to mock with config change

#### 3. High Learning Value

**End-to-End Pattern Validation**
- Validates containerization strategy
- Tests service-to-service communication
- Proves feature flag mechanism
- Demonstrates monitoring and observability

**Resilience Patterns**
- Retry logic (payment may timeout)
- Circuit breaker (payment service down)
- Fallback strategy (revert to mock)
- Timeout handling

**Service Template**
- Establishes project structure
- Defines API conventions
- Sets up health check patterns
- Creates CI/CD template

**Team Learning**
- Low stakes environment for learning
- Mistakes have limited impact
- Builds confidence for harder extractions
- Establishes team practices

#### 4. Clear Demonstration Value

**Observable Difference**
- Call changes from in-process to HTTP
- Network traffic visible in monitoring
- Latency measurable
- Clear before/after comparison

**Easy to Explain**
- Stakeholders understand payment processing
- Clear service boundary
- Obvious benefits (PCI compliance isolation)
- Simple success criteria

**Quick Win**
- Can complete in 2-3 weeks
- Demonstrates progress early
- Builds momentum for migration
- Validates overall approach

#### 5. Future Benefits

**PCI Compliance Preparation**
- Isolates payment processing
- Easier to audit single service
- Can apply stricter security controls
- Reduces compliance scope for other services

**Independent Scaling**
- Can scale payment service independently
- Useful during high-traffic periods (Black Friday)
- Different resource requirements than other services

**Real Gateway Integration**
- Easy to swap mock for real implementation
- Interface already designed for external integration
- No impact on other services

### Why NOT Other Services?

#### Products Service
**Pros**: High business value, clear domain boundary  
**Cons**:
- ❌ Requires database migration or shared database complexity
- ❌ High read volume = performance critical
- ❌ Caching strategy needed from day one
- ❌ Affects every page load (high blast radius)
- ❌ More complex to validate (many edge cases)

**Verdict**: Better as **second** extraction after patterns established

#### Inventory Service
**Pros**: Clear domain boundary, important for scalability  
**Cons**:
- ❌ Requires event infrastructure (message broker)
- ❌ Concurrency control complexity (locking mechanisms)
- ❌ Critical for checkout success (high risk)
- ❌ Database migration required
- ❌ Complex failure scenarios (compensation logic)

**Verdict**: Too complex for first extraction; suitable for **Phase 3**

#### Cart Service
**Pros**: Good isolation, clear API boundaries  
**Cons**:
- ❌ Requires new infrastructure (Redis)
- ❌ Session management complexity
- ❌ Affects user experience directly (high risk)
- ❌ Data migration from SQL to Redis
- ❌ Performance critical (every cart operation)

**Verdict**: Better after Redis infrastructure established; suitable for **Phase 4**

#### Orders Service
**Pros**: Core business domain, high value  
**Cons**:
- ❌ Most complex domain (orchestrates everything)
- ❌ Requires saga pattern (distributed transactions)
- ❌ Dependencies on all other services
- ❌ Database migration required
- ❌ Critical business function (highest risk)

**Verdict**: Should be **last** extraction (Phase 5) after all dependencies extracted

### Comparison Matrix

| Criteria | Payment | Products | Inventory | Cart | Orders |
|----------|---------|----------|-----------|------|--------|
| Database Migration Needed | ✅ No | ❌ Yes | ❌ Yes | ⚠️ Yes (Redis) | ❌ Yes |
| Pre-existing Abstraction | ✅ Yes | ❌ No | ❌ No | ⚠️ Partial | ❌ No |
| Dependencies on Other Services | ✅ None | ✅ None | ⚠️ Events | ⚠️ Products | ❌ All |
| Blast Radius if Failed | ✅ Small | ❌ Large | ❌ Large | ❌ Medium | ❌ Critical |
| Development Complexity | ✅ Low | ⚠️ Medium | ❌ High | ⚠️ Medium | ❌ Very High |
| Rollback Complexity | ✅ Trivial | ⚠️ Medium | ❌ High | ⚠️ Medium | ❌ High |
| Learning Value | ✅ High | ⚠️ Medium | ⚠️ Medium | ⚠️ Medium | ❌ Complex |
| Time to Complete | ✅ 2-3 weeks | ⚠️ 3-4 weeks | ❌ 4-5 weeks | ⚠️ 3-4 weeks | ❌ 5-6 weeks |

✅ = Favorable  ⚠️ = Neutral  ❌ = Unfavorable

**Winner**: Payment Service (most ✅, fewest ❌)

## Implementation Details

### Service Interface

```csharp
// Request
public record PaymentRequest(
    decimal Amount,
    string Currency,
    string Token
);

// Response
public record PaymentResult(
    bool Succeeded,
    string? ProviderRef,
    string? Error
);

// API Endpoint
[HttpPost("/api/payments/charge")]
public async Task<IActionResult> Charge([FromBody] PaymentRequest request)
{
    var result = await _paymentGateway.ChargeAsync(request);
    return Ok(result);
}
```

### Monolith Integration

```csharp
// HTTP Client Implementation
public class HttpPaymentGateway : IPaymentGateway
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<HttpPaymentGateway> _logger;

    public HttpPaymentGateway(HttpClient httpClient, ILogger<HttpPaymentGateway> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<PaymentResult> ChargeAsync(PaymentRequest req, CancellationToken ct = default)
    {
        try
        {
            _logger.LogInformation("Charging payment via Payment Service: {Amount} {Currency}", 
                req.Amount, req.Currency);

            var response = await _httpClient.PostAsJsonAsync("/api/payments/charge", req, ct);
            response.EnsureSuccessStatusCode();

            var result = await response.Content.ReadFromJsonAsync<PaymentResult>(cancellationToken: ct);
            
            _logger.LogInformation("Payment result: {Succeeded}, Ref: {ProviderRef}", 
                result.Succeeded, result.ProviderRef);

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment service call failed");
            throw;
        }
    }
}
```

### Feature Flag Configuration

```csharp
// Program.cs
var usePaymentService = builder.Configuration.GetValue<bool>("FeatureFlags:UsePaymentService");

if (usePaymentService)
{
    builder.Services.AddHttpClient<IPaymentGateway, HttpPaymentGateway>(client =>
    {
        client.BaseAddress = new Uri(builder.Configuration["Services:PaymentService:Url"]!);
        client.Timeout = TimeSpan.FromSeconds(5);
    })
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
}
else
{
    builder.Services.AddScoped<IPaymentGateway, MockPaymentGateway>();
}

// Polly Policies
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
}
```

### Configuration File

```json
{
  "FeatureFlags": {
    "UsePaymentService": false
  },
  "Services": {
    "PaymentService": {
      "Url": "http://payment-service:80"
    }
  }
}
```

## Consequences

### Positive

1. **Quick Validation of Approach**
   - Proves strangler pattern works
   - Validates container deployment
   - Tests service-to-service communication
   - Demonstrates feature flag mechanism

2. **Low-Risk Learning**
   - Team gains microservices experience
   - Mistakes have limited impact
   - Patterns established for future extractions
   - Builds team confidence

3. **Immediate Benefits**
   - Payment processing isolated (security benefit)
   - Independent deployment of payment logic
   - Can scale payment service independently
   - Prepares for real gateway integration

4. **Clear Success Criteria**
   - Payment succeeds = success
   - Payment fails correctly = success
   - Easy to measure and validate
   - Objective demonstration to stakeholders

5. **Template for Future Services**
   - Project structure established
   - API conventions defined
   - Health check patterns proven
   - CI/CD pipeline template created

### Negative

1. **Network Latency**
   - In-process call becomes HTTP call
   - Adds ~20-50ms latency to checkout
   - **Mitigation**: Acceptable for checkout flow (not hot path)

2. **New Failure Modes**
   - Network failures possible
   - Service may be unreachable
   - **Mitigation**: Circuit breaker, retry, fallback to mock

3. **Operational Overhead**
   - Additional service to monitor
   - More containers to manage
   - **Mitigation**: Monitoring tools, runbooks, automation

4. **Limited Business Value**
   - Doesn't directly improve user experience
   - Doesn't add features
   - **Mitigation**: Enables future improvements (real gateway integration)

### Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Payment service unavailable | Low | High | Circuit breaker → fallback to mock, alerts |
| Network latency too high | Low | Medium | Set aggressive timeout (5s), monitor P95 |
| Lost payment requests | Very Low | Critical | Implement idempotency, retry with exponential backoff |
| Incorrect implementation | Low | High | Integration tests, parallel run for validation |
| Team lacks Docker knowledge | Medium | Low | Training, documentation, pair programming |

## Success Criteria

### Functional
- ✅ Payment service processes 100% of payments correctly
- ✅ Successful payments return `Succeeded = true` with provider reference
- ✅ Failed payments return `Succeeded = false` with error message
- ✅ All existing checkout tests pass with service enabled

### Non-Functional
- ✅ P95 latency for payment calls < 200ms
- ✅ Service availability > 99.9%
- ✅ Rollback time < 5 minutes (feature flag toggle)
- ✅ Zero increase in payment failure rate

### Operational
- ✅ Payment service health check reports healthy
- ✅ Logs show correlation IDs across monolith and service
- ✅ Metrics dashboard shows payment success/failure rates
- ✅ Alerts configured for service degradation

## Rollback Plan

If issues arise:

1. **Immediate Rollback** (< 5 minutes)
   - Set `FeatureFlags:UsePaymentService = false` in configuration
   - Restart monolith application
   - Traffic reverts to in-process `MockPaymentGateway`
   - No data loss (no database involved)

2. **Post-Rollback Actions**
   - Investigate logs and metrics
   - Identify root cause
   - Fix issue in Payment Service
   - Re-test before re-enabling

3. **Gradual Re-Enable**
   - Enable for 1% of traffic
   - Monitor for errors
   - Gradually increase to 100%

## Timeline

**Week 1**: Development
- Create Payment Service project
- Implement API endpoint
- Create HttpPaymentGateway in monolith
- Add feature flag logic
- Write integration tests

**Week 2**: Testing and Deployment
- Deploy to dev environment
- Run integration tests
- Deploy to staging environment
- Load testing
- Documentation

**Week 3**: Gradual Rollout
- Enable in production at 1%
- Monitor for 24 hours
- Increase to 10%, then 50%, then 100%
- Validate success criteria
- Document learnings

## Related Decisions

- **ADR-005**: Container-Based Deployment (enables service deployment)
- **ADR-006**: Strangler Fig Migration Pattern (overall migration strategy)
- **ADR-003**: Mock Payment Gateway (interface being extracted)

## Conclusion

Payment Service is the optimal choice for first service extraction because it:

✅ **Minimizes Risk**: No database, simple logic, limited blast radius  
✅ **Maximizes Learning**: End-to-end pattern validation  
✅ **Quick to Implement**: 2-3 weeks to production  
✅ **Easy to Rollback**: Feature flag toggle  
✅ **Clear Success Criteria**: Objective and measurable  
✅ **Sets Patterns**: Template for future extractions  

This decision balances technical simplicity with learning value, making it the ideal starting point for our microservices migration journey.

**Next Step**: Begin implementation of Payment Service in Phase 1 of the migration plan.
