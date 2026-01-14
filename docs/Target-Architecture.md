# Target Architecture - Retail Application Modernization

## Overview

This document describes the target architecture for migrating from the current monolithic Retail Application to a container-based microservices architecture using an incremental, strangler-fig pattern approach.

## Design Principles

1. **Incremental Migration**: Extract services one at a time, maintaining system stability
2. **Behavior Preservation**: No functional changes during extraction
3. **Container-First**: All services run in containers for consistent deployment
4. **API Gateway Pattern**: Central routing and orchestration
5. **Database per Service**: Each service owns its data (eventual goal)
6. **Cloud-Ready**: Architecture supports deployment to cloud platforms (Azure, AWS)

## Proposed Service Boundaries

Based on the logical domain boundaries identified in the monolith (HLD), we propose the following service decomposition:

### 1. Payment Service
**Domain**: Payment Processing  
**Responsibilities**:
- Process payment transactions via payment gateway
- Handle payment confirmations and failures
- Manage payment provider integration

**Current Interface**:
```csharp
IPaymentGateway.ChargeAsync(PaymentRequest, CancellationToken)
  → PaymentResult(Succeeded, ProviderRef, Error)
```

**Why First**: 
- Well-defined interface with single responsibility
- No database dependencies
- Easily reversible extraction
- Clear contract boundary

### 2. Products Service
**Domain**: Product Catalog  
**Responsibilities**:
- Manage product catalog (CRUD operations)
- Product information (SKU, name, description, price, category)
- Product active/inactive status
- Product search and filtering

**Data Ownership**:
- `Products` table (id, sku, name, description, price, currency, isActive, category)

**APIs**:
- `GET /api/products` - List active products
- `GET /api/products/{id}` - Get product details
- `GET /api/products/sku/{sku}` - Get product by SKU
- `POST /api/products` - Create product (admin)
- `PUT /api/products/{id}` - Update product (admin)

### 3. Inventory Service
**Domain**: Stock Management  
**Responsibilities**:
- Track stock levels per SKU
- Reserve inventory during checkout
- Release reservations on cancellation
- Update inventory levels

**Data Ownership**:
- `Inventory` table (id, sku, quantity)

**APIs**:
- `GET /api/inventory/{sku}` - Get stock level
- `POST /api/inventory/reserve` - Reserve inventory
- `POST /api/inventory/release` - Release reservation
- `PUT /api/inventory/{sku}` - Update stock level

**Integration Points**:
- Subscribes to: OrderCreated, OrderCancelled events
- Publishes: InventoryReserved, InventoryInsufficient events

### 4. Cart Service
**Domain**: Shopping Cart  
**Responsibilities**:
- Manage customer shopping carts
- Add/remove/update cart items
- Calculate cart totals
- Clear cart after checkout

**Data Ownership**:
- `Carts` table (id, customerId)
- `CartLines` table (id, cartId, sku, name, unitPrice, quantity)

**APIs**:
- `GET /api/cart/{customerId}` - Get cart with lines
- `POST /api/cart/{customerId}/items` - Add item to cart
- `PUT /api/cart/{customerId}/items/{sku}` - Update cart item
- `DELETE /api/cart/{customerId}/items/{sku}` - Remove cart item
- `DELETE /api/cart/{customerId}` - Clear cart

### 5. Orders Service
**Domain**: Order Management  
**Responsibilities**:
- Create and manage orders
- Track order status (Created, Paid, Failed, Shipped)
- Order history and retrieval
- Orchestrate checkout process

**Data Ownership**:
- `Orders` table (id, createdUtc, customerId, status, total)
- `OrderLines` table (id, orderId, sku, name, unitPrice, quantity)

**APIs**:
- `GET /api/orders` - List orders for customer
- `GET /api/orders/{id}` - Get order details
- `POST /api/orders/checkout` - Process checkout (saga orchestrator)
- `PUT /api/orders/{id}/status` - Update order status

**Integration Points**:
- Calls: Payment Service, Inventory Service
- Publishes: OrderCreated, OrderPaid, OrderFailed events

### 6. Web Frontend (Monolith Facade)
**Responsibilities**:
- Server-side rendered UI (Razor Pages)
- API Gateway / BFF pattern
- Session management
- Route requests to backend services

**Evolution Path**:
- Initially: Razor Pages calling backend services via HTTP
- Future: Could be replaced with React/Vue SPA + API Gateway

## Deployment Model

### Container Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Load Balancer / Ingress                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway / Web Frontend               │
│                    (Nginx / Ocelot / YARP)                  │
└─────────────────────────────────────────────────────────────┘
          │          │          │          │          │
          ▼          ▼          ▼          ▼          ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Payment │ │Products │ │Inventory│ │  Cart   │ │ Orders  │
    │ Service │ │ Service │ │ Service │ │ Service │ │ Service │
    └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
          │          │          │          │          │
          ▼          ▼          ▼          ▼          ▼
    ┌─────────────────────────────────────────────────────────┐
    │                    Data Stores                           │
    │  (Payment: Stateless)  (Products DB)  (Inventory DB)     │
    │  (Cart DB)             (Orders DB)                       │
    └─────────────────────────────────────────────────────────┘
```

### Container Specifications

Each service will be containerized with:
- **Base Image**: `mcr.microsoft.com/dotnet/aspnet:8.0` (runtime)
- **Build Image**: `mcr.microsoft.com/dotnet/sdk:8.0` (multi-stage build)
- **Health Checks**: `/health` endpoint for liveness/readiness probes
- **Configuration**: Environment variables and mounted config files
- **Logging**: Structured logging to stdout (JSON format)
- **Metrics**: Prometheus endpoints for observability

### Orchestration Options

#### Option 1: Docker Compose (Development/Testing)
```yaml
services:
  api-gateway:
    image: retail/api-gateway:latest
    ports: ["80:80"]
    depends_on: [payment-service, products-service, ...]
  
  payment-service:
    image: retail/payment-service:latest
    environment:
      - PaymentProvider__ApiKey=${PAYMENT_API_KEY}
  
  products-service:
    image: retail/products-service:latest
    depends_on: [products-db]
  
  products-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - SA_PASSWORD=${DB_PASSWORD}
```

#### Option 2: Kubernetes (Production)
- **Deployments**: Each service as a Deployment with 2+ replicas
- **Services**: ClusterIP services for internal communication
- **Ingress**: Route external traffic to API Gateway
- **ConfigMaps**: Non-sensitive configuration
- **Secrets**: Database credentials, API keys
- **Persistent Volumes**: For database pods
- **HPA**: Horizontal Pod Autoscaler for scaling

#### Option 3: Azure Container Apps (PaaS)
- **Container Apps**: Each service as a Container App
- **Environment**: Shared Container Apps Environment
- **Ingress**: Built-in ingress controller
- **Scaling**: KEDA-based autoscaling
- **Identity**: Managed identities for Azure resources
- **Secrets**: Azure Key Vault integration

### Database Strategy

#### Phase 1: Shared Database (Strangler Phase)
- Single SQL Server instance
- Logical separation via schemas
- Services access via views/stored procedures
- Gradual data ownership transition

#### Phase 2: Database per Service (Target State)
- Each service has its own database
- Data synchronization via events
- Saga pattern for distributed transactions
- Eventual consistency model

### Service Communication Patterns

#### Synchronous Communication (HTTP/REST)
**When to Use**:
- Request-response patterns
- Real-time data requirements
- Frontend-to-backend calls
- Read operations

**Implementation**:
- RESTful APIs with JSON
- HTTP/2 for performance
- Circuit breaker pattern (Polly)
- Retry with exponential backoff
- Timeout configuration

**Example Flow**:
```
API Gateway → Orders Service → Payment Service (sync)
              ↓
              → Products Service (sync, get product details)
```

#### Asynchronous Communication (Events/Messaging)
**When to Use**:
- Event notifications
- Background processing
- Decoupling services
- Long-running operations

**Implementation Options**:
1. **Azure Service Bus**: Enterprise messaging
2. **RabbitMQ**: Open-source message broker
3. **Azure Event Hubs**: High-throughput streaming
4. **Kafka**: Distributed streaming platform

**Event Examples**:
- `OrderCreated` → Inventory Service reserves stock
- `PaymentCompleted` → Orders Service updates status
- `InventoryReserved` → Orders Service continues checkout

#### Data Access Patterns

**Pattern 1: Direct Database Access (Phase 1)**
- Services access shared database
- Schema per domain for logical separation
- Use views to abstract table structure
- Prepare for eventual data migration

**Pattern 2: API-First (Phase 2+)**
- Services expose APIs for data access
- No direct database access across services
- Data duplication via events (CQRS)
- Eventual consistency

**Pattern 3: CQRS (Optional Enhancement)**
- Command side: Write operations
- Query side: Read-optimized projections
- Event sourcing for audit trail
- Suitable for Orders and Inventory domains

## Configuration Management

### Environment-Specific Configuration

**Development**:
- Local connection strings
- Mock payment gateway
- In-memory caching
- Verbose logging

**Staging**:
- Shared staging databases
- Payment sandbox
- Redis for caching
- Info-level logging

**Production**:
- Production databases with read replicas
- Real payment gateway
- Distributed cache (Redis/Azure Cache)
- Warning/Error logging
- Secrets from Azure Key Vault

### Configuration Sources (Priority Order)
1. Command-line arguments
2. Environment variables
3. Azure Key Vault (production)
4. appsettings.{Environment}.json
5. appsettings.json

### Secret Management
- **Development**: User secrets (`dotnet user-secrets`)
- **CI/CD**: GitHub Secrets or Azure DevOps variables
- **Production**: Azure Key Vault with Managed Identity

## Routing Strategy

### API Gateway Responsibilities
1. **Request Routing**: Route to appropriate backend service
2. **Authentication**: JWT validation, OAuth2
3. **Rate Limiting**: Prevent abuse
4. **Request Transformation**: Adapt legacy clients
5. **Response Aggregation**: Combine multiple service calls
6. **Caching**: Cache frequently accessed data
7. **Logging**: Centralized request logging

### Routing Table Example
```
/api/products/*       → Products Service
/api/inventory/*      → Inventory Service
/api/cart/*           → Cart Service
/api/orders/*         → Orders Service
/api/payments/*       → Payment Service
/                     → Web Frontend (Razor Pages)
```

### Strangler Fig Pattern
During migration:
- **Old Path**: Web Frontend → Monolith (local calls)
- **New Path**: Web Frontend → Service (HTTP calls via API Gateway)
- **Gradual Cutover**: Feature flags control routing
- **Fallback**: Revert to monolith if service fails

## Security Considerations

### Authentication & Authorization
- **Customer Auth**: JWT tokens from identity provider
- **Service-to-Service**: Mutual TLS or API keys
- **Admin Operations**: Role-based access control (RBAC)

### Network Security
- **Container Network**: Private network for inter-service communication
- **TLS Everywhere**: HTTPS for all external traffic
- **Network Policies**: Restrict service-to-service communication

### Data Security
- **Encryption at Rest**: Database encryption
- **Encryption in Transit**: TLS 1.3
- **PCI Compliance**: Payment service isolated and audited
- **Secrets Rotation**: Regular rotation of credentials

## Observability

### Logging
- **Structured Logging**: JSON format with correlation IDs
- **Centralized**: Azure Application Insights or ELK stack
- **Log Levels**: Consistent across services
- **Correlation**: Distributed tracing with correlation IDs

### Monitoring
- **Health Checks**: Liveness and readiness probes
- **Metrics**: CPU, memory, request rate, error rate
- **Dashboards**: Grafana or Azure Monitor
- **Alerts**: On error rate spikes, latency increases

### Distributed Tracing
- **OpenTelemetry**: Standard instrumentation
- **Trace Context**: W3C Trace Context propagation
- **Visualization**: Jaeger or Azure Application Insights

## Scalability & Performance

### Horizontal Scaling
- **Stateless Services**: All services designed stateless
- **Load Balancing**: Round-robin or least connections
- **Session Affinity**: Sticky sessions for cart (if needed)

### Caching Strategy
- **API Gateway**: Cache product listings (1-5 min TTL)
- **Products Service**: Cache product details in Redis
- **Cart Service**: Redis for cart state (session-based)
- **Inventory Service**: Short TTL cache, prioritize accuracy

### Database Optimization
- **Read Replicas**: For read-heavy services (Products)
- **Connection Pooling**: Efficient database connections
- **Indexing**: Optimize queries based on access patterns
- **Partitioning**: For large tables (Orders, CartLines)

## Disaster Recovery

### Backup Strategy
- **Databases**: Daily automated backups
- **Configuration**: Version-controlled in Git
- **Secrets**: Backed up in secure vault
- **Recovery Time Objective (RTO)**: 1 hour
- **Recovery Point Objective (RPO)**: 24 hours

### High Availability
- **Multi-AZ Deployment**: Distribute across availability zones
- **Database Replication**: Synchronous or asynchronous replication
- **Service Redundancy**: Minimum 2 replicas per service
- **Graceful Degradation**: Fallback to cached data if services unavailable

## Cost Optimization

### Resource Sizing
- **Right-Sizing**: Start small, scale based on metrics
- **Burstable Instances**: For variable workloads
- **Spot Instances**: For non-critical workloads

### Database Costs
- **Shared Database**: Initially reduce costs
- **Serverless Databases**: Azure SQL Serverless for low-traffic services
- **Read Replicas**: Only where needed (Products)

### Container Optimization
- **Multi-Stage Builds**: Smaller image sizes
- **Base Image**: Minimal runtime images
- **Resource Limits**: Prevent resource over-allocation

## Migration Readiness

This target architecture supports:
✅ Incremental migration (strangler pattern)  
✅ Container-based deployment  
✅ Behavior preservation during migration  
✅ No big-bang rewrite  
✅ Cloud-ready (Azure, AWS, GCP)  
✅ Observable and monitorable  
✅ Scalable and resilient  

## Next Steps

Refer to `/docs/Migration-Plan.md` for the step-by-step migration phases and timeline.
