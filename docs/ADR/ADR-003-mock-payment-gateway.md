# ADR-003: Mock Payment Gateway for Development

## Status
Accepted

## Context
The checkout process requires integration with a payment processor to charge customers. A decision was needed on how to handle payment processing during development and testing phases.

## Decision
Define a `IPaymentGateway` interface and implement a `MockPaymentGateway` that simulates successful payment processing without calling real payment services.

## Rationale

### Interface-First Design
```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(PaymentRequest req, CancellationToken ct = default);
}
```

This abstraction provides:
1. **Testability**: Easy to mock in unit tests
2. **Flexibility**: Real implementation can be swapped in without changing consuming code
3. **Development Speed**: No need for payment provider credentials during development
4. **Cost Savings**: Avoid sandbox/test transaction fees during development

### Mock Implementation
```csharp
public class MockPaymentGateway : IPaymentGateway
{
    public Task<PaymentResult> ChargeAsync(PaymentRequest req, CancellationToken ct = default)
    {
        return Task.FromResult(new PaymentResult(true, $"MOCK-{Guid.NewGuid():N}", null));
    }
}
```

Characteristics:
- Always returns successful payment
- Generates mock provider reference ID
- No network calls or external dependencies
- Synchronous execution wrapped in completed Task

## Implementation Details

### Records
- `PaymentRequest(decimal Amount, string Currency, string Token)` - Input
- `PaymentResult(bool Succeeded, string? ProviderRef, string? Error)` - Output

### Registration
```csharp
builder.Services.AddScoped<IPaymentGateway, MockPaymentGateway>();
```

Registered as scoped service, matching typical payment gateway usage patterns.

### Usage
Called by `CheckoutService.CheckoutAsync()`:
```csharp
var pay = await _payments.ChargeAsync(new(total, "GBP", paymentToken), ct);
var status = pay.Succeeded ? "Paid" : "Failed";
```

Order status set based on payment result.

## Consequences

### Positive
- **Fast Development**: No payment provider setup required
- **Reliable Testing**: Consistent behavior in tests
- **No External Dependencies**: Can develop offline
- **No Cost**: Free to test repeatedly
- **Interface Stability**: Real implementation won't require changes to CheckoutService

### Negative
- **Incomplete Error Handling**: Failure path never exercised
  - `CheckoutService` assumes failure means "Failed" order status
  - No retry logic tested
  - No timeout handling
  - No network error handling
  - Inventory already reserved on payment failure (not rolled back)

- **False Confidence**: Tests pass but may fail with real gateway
  - Real gateways have latency, rate limits, errors
  - Idempotency requirements not tested
  - PCI compliance not considered

- **Production Risk**: Easy to accidentally deploy with mock
  - No runtime check to ensure real implementation in production
  - Should fail fast if mock used in production environment

- **Limited Validation**: Mock doesn't validate request fields
  - Real gateway would reject invalid amounts, currencies, tokens
  - Invalid inputs not caught until production

### Future Considerations

#### Real Payment Gateway Implementation
A production implementation would need:
1. **Provider Selection**: Stripe, PayPal, Adyen, etc.
2. **Authentication**: API keys, OAuth tokens
3. **Error Handling**: Network failures, timeouts, retries
4. **Idempotency**: Prevent duplicate charges on retries
5. **Webhook Handling**: Async payment confirmations
6. **PCI Compliance**: Secure token handling
7. **Logging**: Payment attempts, failures, provider responses
8. **Monitoring**: Success rates, latency, error types

#### Testing Strategy
- Keep mock for unit tests
- Use payment provider sandbox for integration tests
- Use feature flags to control implementation selection
- Consider test mode detection in production deployments

#### Configuration Example
```csharp
if (builder.Environment.IsProduction())
{
    builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
}
else
{
    builder.Services.AddScoped<IPaymentGateway, MockPaymentGateway>();
}
```

## Known Issues

1. **No Failure Testing**: Cannot test checkout failure path without modifying mock
   - Could add configuration to simulate failures
   - Could add test-specific failure injection

2. **Hardcoded Success**: Mock always succeeds
   - Real-world failures never exercised
   - Error handling code paths untested

3. **Inventory Reservation Issue**: Combined with checkout transaction design, payment failure leaves inventory reserved
   - Addressed in checkout service design, not payment gateway
   - Real implementation might use pre-authorization then capture pattern

4. **Currency Hardcoded**: "GBP" used throughout
   - Real gateway would validate currency support
   - Multi-currency not considered

## Alternatives Considered

### Payment Provider Sandbox
- **Pros**: More realistic testing, tests integration
- **Cons**: Requires credentials, network dependency, slower tests
- **Verdict**: Use sandbox for integration tests, keep mock for unit tests

### In-Memory Queue + Background Processing
- **Pros**: Simulates async payment processing
- **Cons**: Over-engineering for current needs
- **Verdict**: YAGNI - implement when needed

### No Abstraction (Direct Provider SDK)
- **Pros**: Fewer layers, more direct
- **Cons**: Tight coupling, hard to test, hard to switch providers
- **Verdict**: Abstraction provides value for testing and flexibility

## Related Decisions
- ADR-001: Monolithic architecture (synchronous payment processing acceptable)
- Checkout Service Transaction Design (impacts payment failure handling)
