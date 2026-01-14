# ADR-005: Container-Based Deployment Strategy

## Status
Accepted

## Context

The application is currently deployed as a monolithic ASP.NET Core application with dependencies on SQL Server. As we migrate to a microservices architecture, we need a consistent deployment model that:

1. Supports multiple independent services
2. Enables easy local development and testing
3. Facilitates continuous deployment
4. Provides environment consistency (dev, staging, production)
5. Supports scalability and resource management
6. Is compatible with modern cloud platforms (Azure, AWS, GCP)

## Decision

Adopt **container-based deployment** using Docker as the primary containerization technology, with support for orchestration via Docker Compose (development/testing) and Kubernetes or Azure Container Apps (production).

## Rationale

### Why Containers

1. **Consistency Across Environments**
   - "It works on my machine" problem eliminated
   - Same container image runs in dev, staging, and production
   - OS-level dependencies packaged with application
   - Eliminates configuration drift

2. **Microservices-Friendly**
   - Each service runs in its own container
   - Independent deployment and versioning
   - Isolated dependencies (no version conflicts)
   - Easy to scale individual services

3. **Developer Productivity**
   - Fast startup times compared to VMs
   - Easy to spin up entire stack locally (docker-compose up)
   - Simplified onboarding for new developers
   - Consistent tooling across team

4. **CI/CD Integration**
   - Build once, deploy anywhere
   - Container images are immutable artifacts
   - Easy rollback (deploy previous image)
   - Automated build and push to registry

5. **Resource Efficiency**
   - Lightweight compared to VMs (no OS overhead)
   - Fast startup and shutdown
   - Higher density on host machines
   - Cost-effective scaling

6. **Cloud-Native**
   - First-class support in all major cloud providers
   - Azure Container Apps, AWS ECS/Fargate, GCP Cloud Run
   - Kubernetes (AKS, EKS, GKE) for orchestration
   - Serverless container options (Azure Container Instances)

### Why Docker Specifically

1. **Industry Standard**: Most widely adopted container platform
2. **Mature Ecosystem**: Extensive tooling, documentation, community support
3. **Multi-Stage Builds**: Optimize image sizes (separate build and runtime images)
4. **Docker Compose**: Simplifies multi-container local development
5. **Registry Support**: Docker Hub, Azure Container Registry, AWS ECR, GitHub Container Registry
6. **Windows Container Support**: Can containerize Windows-based .NET Framework apps if needed

### Orchestration Strategy

#### Development: Docker Compose
- Simple `docker-compose.yml` defines all services
- Single command to start/stop entire stack
- Suitable for local development and integration testing
- No additional infrastructure required

**Example**:
```yaml
version: '3.8'
services:
  monolith:
    build: .
    ports: ["5000:80"]
    environment:
      - ConnectionStrings__DefaultConnection=Server=db;...
    depends_on:
      - db
  
  payment-service:
    build: ./src/PaymentService
    ports: ["5001:80"]
    environment:
      - PaymentProvider__ApiKey=${PAYMENT_API_KEY}
  
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - SA_PASSWORD=${DB_PASSWORD}
      - ACCEPT_EULA=Y
    volumes:
      - dbdata:/var/opt/mssql
```

#### Production: Kubernetes or Azure Container Apps

**Option 1: Kubernetes (AKS, EKS, GKE)**
- Suitable for complex deployments with many services
- Full control over networking, scaling, service mesh
- Requires Kubernetes expertise
- Higher operational overhead
- Best for: Large-scale, multi-service architectures

**Option 2: Azure Container Apps (Recommended for This Project)**
- Managed Kubernetes under the hood (abstracted)
- Simpler deployment model (no YAML complexity)
- Built-in scaling (KEDA-based)
- Integrated with Azure services (Key Vault, App Insights)
- Lower operational overhead
- Best for: Small to medium microservices deployments

**Decision for This Project**: Start with **Azure Container Apps** for production due to:
- Simpler operational model
- Built-in observability (Azure Monitor)
- Cost-effective for low to medium traffic
- Can migrate to full AKS later if needed

## Implementation Details

### Dockerfile Structure (Multi-Stage Build)

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["RetailMonolith.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
EXPOSE 80
EXPOSE 443
ENTRYPOINT ["dotnet", "RetailMonolith.dll"]
```

**Benefits**:
- Final image contains only runtime dependencies (~200 MB)
- Build artifacts not included in runtime image (security)
- Faster image pulls and deployments
- Consistent across all services

### Container Configuration Best Practices

1. **Environment Variables for Configuration**
   - No hardcoded secrets in images
   - Runtime configuration via environment variables
   - Secrets from Azure Key Vault or Kubernetes Secrets

2. **Health Checks**
   - Dockerfile HEALTHCHECK instruction
   - HTTP endpoint `/health` for liveness/readiness probes
   - Ensures container is ready to receive traffic

3. **Logging to stdout/stderr**
   - Container logs captured by orchestration platform
   - Structured logging (JSON format)
   - Centralized log aggregation (Azure Log Analytics, ELK)

4. **Non-Root User**
   - Run application as non-root user for security
   - Principle of least privilege

5. **Image Tagging Strategy**
   - Semantic versioning: `v1.2.3`
   - Git commit SHA: `abc1234`
   - Environment: `latest` (dev only), `stable` (production)
   - Never use `latest` in production

### Container Registry

**Recommendation**: Azure Container Registry (ACR)
- Private registry for container images
- Integrated with Azure services (AKS, Container Apps)
- Vulnerability scanning (Azure Defender)
- Geo-replication for high availability
- Cost-effective for small teams

**Alternative**: GitHub Container Registry (GHCR)
- Free for public repositories
- Integrated with GitHub Actions CI/CD
- Good for open-source projects

## Consequences

### Positive

1. **Simplified Deployment**
   - Single artifact (container image) to deploy
   - Consistent deployment process across all services
   - Rollback is simple (deploy previous image version)

2. **Environment Parity**
   - Dev, staging, production use same images
   - Eliminates "works in dev, fails in production" issues
   - Configuration differences isolated to environment variables

3. **Rapid Scaling**
   - Horizontal scaling by running more container instances
   - Fast startup times enable quick scale-up
   - Orchestration handles load balancing automatically

4. **Improved Isolation**
   - Each service runs in isolated environment
   - Dependency conflicts eliminated
   - Failures contained within containers

5. **Developer Experience**
   - Simple local environment setup (docker-compose up)
   - Fast iteration cycles
   - Consistent tooling across team

### Negative

1. **Learning Curve**
   - Team needs Docker knowledge
   - Debugging inside containers requires new skills
   - Orchestration platforms (Kubernetes) have steep learning curve
   - **Mitigation**: Training, documentation, start with Docker Compose

2. **Operational Complexity**
   - Additional infrastructure to manage (registry, orchestration)
   - Container health monitoring required
   - Log aggregation across containers needed
   - **Mitigation**: Use managed services (Azure Container Apps, ACR)

3. **Increased Build Time**
   - CI/CD builds container images (slower than direct deployment)
   - Image pulls add deployment time
   - **Mitigation**: Multi-stage builds, layer caching, use fast registry

4. **Storage and Networking**
   - Persistent storage requires volume management
   - Container networking adds abstraction layer
   - DNS resolution for service discovery
   - **Mitigation**: Use orchestration platform features (Kubernetes PVs, Services)

5. **Windows Container Limitations** (if applicable)
   - Larger image sizes compared to Linux containers
   - Fewer base images available
   - Windows Server license costs
   - **Mitigation**: Use Linux containers for ASP.NET Core (supported)

### Known Issues

1. **LocalDB Not Supported in Containers**
   - LocalDB requires Windows user profile
   - Not available in Docker containers
   - **Solution**: Use SQL Server container for development
   - Already compatible: Connection string configurable via environment variable

2. **Multi-Stage Build Cache**
   - Layer caching can cause stale builds if not careful
   - **Solution**: Order Dockerfile instructions from least to most frequently changing

3. **Development Environment Differences**
   - macOS/Linux developers using Linux containers
   - Windows developers may use Windows or Linux containers
   - **Solution**: Use Linux containers for cross-platform consistency

## Migration Path

### Phase 1: Containerize Monolith (Week 1-2)
1. Create Dockerfile for RetailMonolith
2. Create docker-compose.yml with monolith + SQL Server
3. Test full application in containers
4. Update CI/CD to build container images

### Phase 2: Add Services Incrementally (Week 3+)
1. Each new service gets its own Dockerfile
2. Add service to docker-compose.yml
3. Test locally with docker-compose
4. Deploy to Azure Container Apps

### Phase 3: Production Orchestration (Week 8+)
1. Deploy to Azure Container Apps environment
2. Configure scaling rules
3. Set up monitoring and logging
4. Implement blue-green deployments

## Alternatives Considered

### Virtual Machines (VMs)
- **Pros**: Familiar deployment model, full OS control
- **Cons**: Heavy resource usage, slow startup, manual configuration drift
- **Verdict**: Not suitable for microservices (too heavyweight)

### Platform-as-a-Service (Azure App Service)
- **Pros**: Simple deployment, managed infrastructure, built-in scaling
- **Cons**: Less control, harder to run multiple services, vendor lock-in
- **Verdict**: Good for monolith, but containers better for microservices transition

### Serverless Functions (Azure Functions)
- **Pros**: Auto-scaling, pay-per-execution, no infrastructure management
- **Cons**: Cold start latency, limited execution time, not suited for stateful services
- **Verdict**: Not appropriate for full application services, but could use for specific functions

### Bare Metal / Direct Deployment
- **Pros**: Maximum performance, no containerization overhead
- **Cons**: Manual environment setup, dependency conflicts, no isolation
- **Verdict**: Not suitable for modern microservices architecture

## Related Decisions

- **ADR-001**: Monolithic Architecture (now migrating to microservices)
- **ADR-006**: Strangler Fig Migration Pattern (incremental service extraction)
- **Migration Plan**: Phased approach to containerization and service extraction

## References

- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Azure Container Apps Documentation](https://learn.microsoft.com/en-us/azure/container-apps/)
- [.NET Docker Images](https://hub.docker.com/_/microsoft-dotnet)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)

## Appendix: Sample Dockerfile for .NET Service

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Copy csproj and restore dependencies (cached layer)
COPY ["ServiceName/ServiceName.csproj", "ServiceName/"]
RUN dotnet restore "ServiceName/ServiceName.csproj"

# Copy source and build
COPY . .
WORKDIR "/src/ServiceName"
RUN dotnet build "ServiceName.csproj" -c $BUILD_CONFIGURATION -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "ServiceName.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

# Copy published app
COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl --fail http://localhost:80/health || exit 1

EXPOSE 80
ENTRYPOINT ["dotnet", "ServiceName.dll"]
```
