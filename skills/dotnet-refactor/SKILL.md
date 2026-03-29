---
name: dotnet-refactor
description: Use when refactoring .NET Core code, reviewing architecture, or checking if code follows James's style in C# projects. Triggers on: private methods, God Class, DI lifetime issues, impure service layer, mutable variables, scattered model conversions, if/switch branch implementations.
---

# .NET Core Refactor Style

## Rules at a Glance

| Rule | Description |
|------|-------------|
| No private method | Any occurrence is a violation |
| No private static method | Any occurrence is a violation |
| Method <= 5 responsibilities | Count responsibilities, not lines |
| Parameters > 2 | Must extract into a DTO record |
| Complex pre-conditions | Decorator pattern |
| Conditional branch implementations | Strategy pattern |
| Service/Domain layer | Pure code — no package/framework dependencies |
| Business logic | Fluent method chaining |
| Variable assignment | Declare and assign in one statement — no deferred assignment |
| Model conversions | Belong on the source model as `To...()` |

---

## DI Lifetime Rules

### Choosing a Lifetime

| Lifetime | When to use | Notes |
|----------|-------------|-------|
| **Transient** | Lightweight, stateless services | Created on every resolve — higher memory pressure, use sparingly |
| **Scoped** | State that must persist within a single request | Most common choice; standard for DbContext |
| **Singleton** | Application-wide state: config, logging, caching | Memory leaks accumulate over time |

### Injection Rules (violations)

```
❌ Never inject Scoped or Transient into a Singleton
   → Effectively promotes the dependency to Singleton lifetime, causing unexpected shared state

❌ Never inject Transient into Scoped
   → Effectively promotes the Transient to Scoped lifetime
```

```csharp
// WRONG — Scoped injected into Singleton, Scoped becomes Singleton
public class MySingleton {
    public MySingleton(IScopedService scoped) { ... } // violation
}

// RIGHT — use IServiceScopeFactory to create a scope manually
public class MySingleton {
    private readonly IServiceScopeFactory _factory;
    public MySingleton(IServiceScopeFactory factory) => _factory = factory;

    public async Task DoWork() {
        using var scope = _factory.CreateScope();
        var scoped = scope.ServiceProvider.GetRequiredService<IScopedService>();
        await scoped.Process();
    }
}
```

---

## Rules in Detail

### 1. No Private Methods

A `private` method signals one of two problems: the class is doing too much, or the method is too granular to justify its own name.

```csharp
// WRONG
public class OrderService {
    public void Process(Order order) { Validate(order); Save(order); }
    private void Validate(Order order) { ... } // violation
}

// RIGHT — if it has independent meaning, extract a class
public class OrderValidator {
    public void Validate(Order order) { ... }
}

// RIGHT — if it's a single expression, inline it
public void Process(Order order) {
    if (!order.IsValid()) throw new ValidationException();
    Save(order);
}
```

### 2. No Private Static Methods

`private static` means the method has no relationship with its containing class.

```csharp
// WRONG
private static string FormatAddress(Address address) { ... } // violation

// RIGHT — move it into Address
public class Address {
    public string Format() => $"{Street}, {City}";
}
```

### 3. Method Has at Most 5 Responsibilities

Count by responsibility or step, not lines. One query = 1, one transformation = 1, one domain call = 1.

```csharp
// WRONG — seven responsibilities
public async Task<Result> PlaceOrder(CreateOrderDto dto) {
    ValidateDto(dto);              // 1
    var user = GetUser(dto);       // 2
    var products = GetProducts();  // 3
    var discount = CalcDiscount(); // 4
    var order = BuildOrder();      // 5
    await SaveOrder(order);        // 6 ❌
    await SendEmail(order);        // 7 ❌
    return Result.Ok(order);
}

// RIGHT — move validation to a Decorator, core stays <= 5
public async Task<Result> PlaceOrder(CreateOrderDto dto) {
    var user = await _userRepo.Get(dto.UserId);               // 1
    var products = await _productRepo.GetMany(dto.ProductIds); // 2
    var order = BuildOrder(user, products, dto);               // 3
    await _orderRepo.Save(order);                              // 4
    return Result.Ok(order.ToDto());                           // 5
}
```

### 4. Parameters > 2 → DTO Record

```csharp
// WRONG
public Order Build(User user, List<Product> products, Coupon coupon) { ... }

// RIGHT
public record BuildOrderDto(User User, List<Product> Products, Coupon? Coupon);
public Order Build(BuildOrderDto dto) { ... }
```

### 5. Complex Pre-conditions → Decorator

Validation, logging, auth, and other cross-cutting concerns belong in a Decorator. The core class stays focused on business logic.

```csharp
public interface IOrderService {
    Task<Result> PlaceOrder(CreateOrderDto dto);
}

// Core — pure business logic
public class OrderService : IOrderService { ... }

// Decorator — handles validation
public class ValidatedOrderService : IOrderService {
    private readonly IOrderService _inner;
    public ValidatedOrderService(IOrderService inner) => _inner = inner;

    public async Task<Result> PlaceOrder(CreateOrderDto dto) {
        if (!dto.IsValid()) throw new ValidationException(dto.Errors());
        return await _inner.PlaceOrder(dto);
    }
}
```

DI registration:
```csharp
services.AddScoped<OrderService>();
services.AddScoped<IOrderService>(sp =>
    new ValidatedOrderService(sp.GetRequiredService<OrderService>()));
```

### 6. Conditional Branch Implementations → Strategy

No if/switch to dispatch different implementations. Use Strategy.

```csharp
// WRONG
public decimal CalcShipping(Order order) {
    if (order.ShippingType == "Express") return order.Total * 0.1m;
    if (order.ShippingType == "Standard") return 15m;
    throw new NotSupportedException();
}

// RIGHT
public interface IShippingStrategy {
    decimal Calculate(Order order);
}
public class ExpressShipping : IShippingStrategy {
    public decimal Calculate(Order order) => order.Total * 0.1m;
}
public class StandardShipping : IShippingStrategy {
    public decimal Calculate(Order order) => 15m;
}
```

### 7. Pure Code Layer

Service and Domain layers must not import any packages or frameworks.

```csharp
// WRONG
using Microsoft.EntityFrameworkCore;
public class OrderService {
    private readonly AppDbContext _db; // violation — direct ORM dependency
}

// RIGHT — isolate behind a Repository interface
public interface IOrderRepository {
    Task<Order> Get(Guid id);
    Task Save(Order order);
}
public class OrderService {
    private readonly IOrderRepository _repo; // pure — depends only on the interface
}

// Repository implementation (framework dependencies allowed here)
public class EfOrderRepository : IOrderRepository {
    private readonly AppDbContext _db;
    ...
}
```

### 8. Let the Model Speak (To... conversions)

Model conversions belong on the source model as instance methods.

```csharp
// WRONG — conversions scattered across the codebase
var dto = new OrderDto { Id = order.Id, Total = order.Total };

// RIGHT
public class Order {
    public OrderDto ToDto() => new(Id, Total, Status.ToString());
    public OrderSummary ToSummary() => new(Id, CreatedAt, Status);
    public OrderCreatedEvent ToCreatedEvent() => new(Id, UserId, Total);
}
var dto = order.ToDto();
```

### 9. Fluent Business Logic

```csharp
// WRONG
var order = new Order();
order.SetUser(user);
order.AddProducts(products);
order.Confirm();

// RIGHT
var order = Order.For(user)
    .WithProducts(products)
    .ApplyCoupon(coupon)
    .Confirm();
```

### 10. No Deferred Variable Assignment

Declare and assign in one statement. No declare-then-assign patterns.

```csharp
// WRONG
decimal total;
if (hasDiscount) total = price * 0.9m;
else total = price;

// RIGHT
var total = hasDiscount ? price * 0.9m : price;

// WRONG
List<Product> available = new();
foreach (var p in products)
    if (p.IsAvailable) available.Add(p);

// RIGHT
var available = products.Where(p => p.IsAvailable).ToList();

// WRONG
string message;
switch (status) {
    case Status.Active: message = "Active"; break;
}

// RIGHT
var message = status switch {
    Status.Active => "Active",
    Status.Inactive => "Inactive",
    _ => throw new NotSupportedException()
};
```

---

## Refactoring Checklist

1. Scan for `private` / `private static` → resolve all violations first
2. Count method responsibilities → split any method exceeding 5
3. Check parameter counts → extract DTO record for anything over 2
4. Identify complex pre-conditions → extract Decorator
5. Identify conditional branch implementations → extract Strategy
6. Verify layer purity → remove illegal `using` directives
7. Find model conversions → move to source model as `To...()`
8. Apply fluent style to business logic → method chaining
9. Eliminate deferred assignments → ternary, LINQ, switch expression
10. Verify DI lifetimes → no lifetime pollution

## Violation Quick Reference

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| `private void/T Method()` | Too much or too little responsibility | Extract class or inline |
| `private static T Method(SomeObj obj)` | Method belongs to `SomeObj` | Move into `SomeObj` |
| Method exceeds 5 responsibilities | Unclear ownership | Split method or extract Decorator |
| 3+ parameters | Parameter cluster | Extract DTO record |
| Large if/else with different implementations | Missing Strategy | `IStrategy` + implementation classes |
| Service imports framework packages | Layer pollution | Extract `IRepository` interface |
| Variable declared then assigned later | Mutable mindset | Ternary / LINQ / switch expression |
| Model conversions scattered across codebase | Conversions misplaced | Move to `SourceModel.ToXxx()` |
| Scoped injected into Singleton | DI lifetime pollution | Use `IServiceScopeFactory` |
| Transient injected into Scoped | DI lifetime pollution | Adjust lifetime or redesign |
