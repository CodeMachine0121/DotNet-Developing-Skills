---
name: dotnet-architecture
description: Use when designing or reviewing .NET Core project architecture, layer responsibilities, dependency direction, type boundaries, error handling strategy, or placement of cross-cutting concerns such as middleware and action filters.
---

# .NET Core Layered Architecture

## Layer Overview

```
Controller
    ↓ depends on
Application Service
    ↓ depends on
Domain Service  ←→  Repository / Proxy
```

Dependencies flow **inward only**. Inner layers must not reference outer layers.
Interfaces are **defined in inner layers**, implemented in outer layers (Infrastructure).

---

## Layer Responsibilities

### Controller
- Extracts parameters from the HTTP request
- Assembles the input DTO to pass into Application Service
- Maps the result to a **ViewModel** (Response) for the HTTP response
- No business logic. No direct dependency on Repository, Domain Service, or any infrastructure.

### Application Service
- Orchestration only — coordinates calls between Domain Services and/or Repositories
- Receives a **DTO** as input
- Returns a **domain model** when the operation represents a domain event; returns a **DTO** otherwise
- Contains no business rules — any conditional logic on domain state belongs in Domain Service

**Two valid dependency patterns:**

| Scenario | Allowed dependencies |
|----------|----------------------|
| Simple CRUD / query | Application Service → Repository or Proxy directly |
| Domain event handling | Application Service → Domain Service(s) only |

### Domain Service
- Owns business rules and domain logic
- Receives and returns **domain models**
- May depend on Repository or Proxy interfaces (defined here, implemented in Infrastructure)

### Repository
- Abstracts persistence
- Always returns an **Aggregate** — individual entities must not leak out of the infrastructure layer
- Interface defined in Domain layer, implementation in Infrastructure

### Proxy
- Abstracts external APIs and third-party services
- Always returns a **DTO**
- Interface defined in Application or Domain layer, implementation in Infrastructure

---

## Type Boundaries

| Layer boundary | Input type | Output type |
|----------------|-----------|-------------|
| HTTP → Controller | HTTP request | — |
| Controller → Application Service | DTO | — |
| Application Service → caller (Controller) | — | Domain model (domain event) or DTO (query/simple op) |
| Controller → HTTP response | — | ViewModel (converted from domain model) |
| Domain Service → Application Service | Domain model | Domain model |
| Repository → any caller | — | Aggregate |
| Proxy → any caller | — | DTO |

### Key rules
- **ViewModel** (Request/Response) lives in the Controller layer. It is only converted to/from a domain model — never constructed from raw fields scattered across layers.
- **Entities** must not be returned by any public layer boundary. Repository exposes Aggregates; everything else uses domain models or DTOs.
- **DTOs** are passed into Application Service and returned by Proxy. They do not carry domain behavior.

---

## Dependency Direction

```
Controller          →  IApplicationService
ApplicationService  →  IRepository | IProxy       (simple CRUD/query)
ApplicationService  →  IDomainService              (domain event)
DomainService       →  IRepository | IProxy
```

- Interfaces are **defined in the inner layer** that needs them
- Implementations live in Infrastructure
- No layer may import a namespace from a layer above it

---

## Cross-cutting Concerns

### Action Filter — API pre-conditions

Use Action Filters for concerns that apply before a controller action executes and are specific to HTTP/API context:
- Authentication / authorization checks
- Request-level validation (model state)
- Rate limiting, idempotency key checks

```csharp
public class RequireFeatureFlagFilter : IActionFilter {
    public void OnActionExecuting(ActionExecutingContext context) {
        if (!_featureFlags.IsEnabled("NewCheckout"))
            context.Result = new StatusCodeResult(503);
    }
    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

### Middleware — global cross-cutting behavior

Use Middleware for concerns that span the entire request pipeline:
- **Global exception handling** (see below)
- Logging / tracing
- Correlation ID injection
- Response compression, CORS

---

## Error Handling

Each layer is responsible only for **throwing** the appropriate exception. Catching and converting to HTTP responses is handled by a single global exception-handling middleware.

### Exception conventions by layer

| Exception | Thrown by | HTTP status |
|-----------|-----------|-------------|
| `NotFoundException` | Repository, Domain Service | 404 |
| `ValidationException` | Domain Service, Application Service | 422 |
| `UnauthorizedException` | Domain Service | 403 |
| Unhandled `Exception` | anywhere | 500 |

### Global exception middleware

```csharp
public class ExceptionHandlingMiddleware {
    public async Task InvokeAsync(HttpContext context, RequestDelegate next) {
        try {
            await next(context);
        }
        catch (NotFoundException ex) {
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(new { error = ex.Message });
        }
        catch (ValidationException ex) {
            context.Response.StatusCode = 422;
            await context.Response.WriteAsJsonAsync(new { error = ex.Message });
        }
        catch (Exception ex) {
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { error = "Unexpected error." });
        }
    }
}
```

Layers throw — middleware catches. No try/catch blocks inside business logic.

---

## Full Example — Simple Query

```csharp
// Controller
[HttpGet("{id}")]
public async Task<OrderResponse> GetOrder(Guid id) {
    var dto = await _orderAppService.GetOrder(id);
    return OrderResponse.From(dto);           // ViewModel converted from DTO
}

// Application Service (simple query — depends on Repository directly)
public async Task<OrderDto> GetOrder(Guid id) {
    var aggregate = await _orderRepo.Get(id); // Repository returns Aggregate
    return aggregate.ToDto();
}
```

## Full Example — Domain Event

```csharp
// Controller
[HttpPost]
public async Task<OrderResponse> PlaceOrder(PlaceOrderRequest request) {
    var dto = request.ToDto();
    var order = await _orderAppService.PlaceOrder(dto);   // returns domain model
    return OrderResponse.From(order);                     // ViewModel from domain model
}

// Application Service (domain event — depends on Domain Service only)
public async Task<Order> PlaceOrder(PlaceOrderDto dto) {
    return await _orderDomainService.PlaceOrder(dto);
}

// Domain Service (business rules, depends on Repository interface)
public async Task<Order> PlaceOrder(PlaceOrderDto dto) {
    var user = await _userRepo.Get(dto.UserId);
    var products = await _productRepo.GetMany(dto.ProductIds);
    var order = Order.For(user).WithProducts(products).Confirm();
    await _orderRepo.Save(order);
    return order;
}
```

---

## Violation Quick Reference

| Symptom | Fix |
|---------|-----|
| Controller calls Repository directly | Route through Application Service |
| Application Service contains `if` on domain state | Move logic to Domain Service |
| Application Service for domain event calls Repository directly | Route through Domain Service |
| Entity returned from Repository | Wrap in Aggregate |
| Entity exposed at Controller boundary | Convert to ViewModel first |
| ViewModel constructed from raw fields in Service | Move conversion to domain model `To...()` |
| Business logic in Controller | Move to Domain Service |
| Try/catch in a Service or Domain class | Remove — let middleware handle it |
| API pre-condition logic inside Controller action | Move to Action Filter |
| Infrastructure concern (logging, CORS) in Controller | Move to Middleware |
| Inner layer imports outer layer namespace | Invert dependency — define interface in inner layer |
