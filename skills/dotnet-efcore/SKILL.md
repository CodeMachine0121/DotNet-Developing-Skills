---
name: dotnet-efcore
description: Use when implementing or reviewing EF Core usage in .NET Core projects. Covers DbContext setup, Repository implementation, Unit of Work pattern, query optimization, migration management, and common pitfalls.
---

# EF Core Best Practices

## Rules at a Glance

| Rule | Description |
|------|-------------|
| EF only in Repository | No EF references in Service or Domain layers |
| Scoped DbContext | Always register as Scoped |
| Disable lazy loading | Explicit loading only |
| `AsNoTracking()` on reads | All read-only queries must opt out of change tracking |
| No `IQueryable` outside Repository | Never leak query composition across layer boundary |
| `SaveChanges` via Unit of Work only | Repositories only stage changes — UoW commits |
| One migration per logical change | Named after business intent, not timestamps |
| No auto-migrate — ever | `MigrateAsync()` and `EnsureCreatedAsync()` are unconditionally banned |

---

## 1. DbContext Setup

```csharp
// Registration — always Scoped
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("Default"))
           .UseLazyLoadingProxies(false)); // disable lazy loading explicitly
```

### Entity Configuration

Use `IEntityTypeConfiguration<T>` — never Data Annotations on domain models.

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Status)
               .HasConversion<string>()
               .IsRequired();
        builder.OwnsOne(o => o.ShippingAddress); // value object as owned entity
        builder.HasMany(o => o.Items)
               .WithOne()
               .HasForeignKey("OrderId");
    }
}

// Register all configurations in one call
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

---

## 2. Repository Implementation

EF Core must not appear outside the Repository. Return Aggregates — never raw entities or `IQueryable`.

```csharp
public class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    // Read — always AsNoTracking
    public async Task<Order?> Get(Guid id) =>
        await _db.Orders
                 .AsNoTracking()
                 .Include(o => o.Items)
                 .FirstOrDefaultAsync(o => o.Id == id);

    // Read with projection — avoid loading unused columns
    public async Task<IReadOnlyList<OrderSummary>> GetSummaries(Guid userId) =>
        await _db.Orders
                 .AsNoTracking()
                 .Where(o => o.UserId == userId)
                 .Select(o => new OrderSummary(o.Id, o.CreatedAt, o.Status))
                 .ToListAsync();

    // Write — stage only, no SaveChanges here
    public void Add(Order order) => _db.Orders.Add(order);
    public void Remove(Order order) => _db.Orders.Remove(order);
}
```

**Never return `IQueryable`** — it leaks query composition responsibility outside the Repository.

```csharp
// WRONG
public IQueryable<Order> Query() => _db.Orders; // infrastructure leaks out

// RIGHT
public async Task<IReadOnlyList<Order>> GetByStatus(OrderStatus status) =>
    await _db.Orders
             .AsNoTracking()
             .Where(o => o.Status == status)
             .ToListAsync();
```

---

## 3. Unit of Work

`IUnitOfWork` is defined in the Application layer. Repositories only stage changes (`Add`, `Remove`, entity mutation). The Application Service controls when changes are committed.

### Interface (Application layer)

```csharp
public interface IUnitOfWork
{
    Task CommitAsync(CancellationToken ct = default);
}
```

### Implementation (Infrastructure layer)

```csharp
public class EfUnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _db;
    public EfUnitOfWork(AppDbContext db) => _db = db;

    public async Task CommitAsync(CancellationToken ct = default) =>
        await _db.SaveChangesAsync(ct);
}
```

### Registration

```csharp
services.AddScoped<IUnitOfWork, EfUnitOfWork>();
```

### Usage in Application Service

```csharp
public class OrderAppService
{
    private readonly IOrderRepository _orderRepo;
    private readonly IUnitOfWork _uow;

    public OrderAppService(IOrderRepository orderRepo, IUnitOfWork uow)
    {
        _orderRepo = orderRepo;
        _uow = uow;
    }

    public async Task PlaceOrder(PlaceOrderDto dto)
    {
        var order = Order.For(dto.UserId).WithProducts(dto.ProductIds).Confirm();
        _orderRepo.Add(order);
        await _uow.CommitAsync(); // single commit for all staged changes
    }
}
```

### Multiple repositories in one transaction

```csharp
public async Task TransferStock(TransferStockDto dto)
{
    var source = await _warehouseRepo.Get(dto.SourceId);
    var target = await _warehouseRepo.Get(dto.TargetId);

    source.Deduct(dto.ProductId, dto.Quantity);
    target.Receive(dto.ProductId, dto.Quantity);

    await _uow.CommitAsync(); // both changes committed atomically
}
```

---

## 4. Query Optimization

### AsNoTracking — all read-only queries

```csharp
// WRONG — change tracker overhead on data that won't be modified
var orders = await _db.Orders.ToListAsync();

// RIGHT
var orders = await _db.Orders.AsNoTracking().ToListAsync();
```

### Select projection — load only what is needed

```csharp
// WRONG — loads entire entity including navigation properties
var names = await _db.Users.Select(u => u.Name).ToListAsync(); // still ok but...

// WRONG — loads full Order to get only Id and Status
var orders = await _db.Orders.AsNoTracking().ToListAsync();
var summaries = orders.Select(o => new OrderSummary(o.Id, o.Status));

// RIGHT — project in the query
var summaries = await _db.Orders
    .AsNoTracking()
    .Select(o => new OrderSummary(o.Id, o.Status))
    .ToListAsync();
```

### N+1 — use Include or AsSplitQuery

```csharp
// WRONG — N+1: one query per order to load items
var orders = await _db.Orders.AsNoTracking().ToListAsync();
foreach (var o in orders) { var items = o.Items; } // triggers N queries

// RIGHT — eager load with Include
var orders = await _db.Orders
    .AsNoTracking()
    .Include(o => o.Items)
    .ToListAsync();
```

### Cartesian explosion — use AsSplitQuery for multiple Includes

```csharp
// WRONG — multiple collection Includes produce a cartesian product
var orders = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments) // cartesian explosion if both are collections
    .ToListAsync();

// RIGHT — split into separate queries
var orders = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery()
    .ToListAsync();
```

### Pagination

```csharp
public async Task<IReadOnlyList<Order>> GetPaged(int page, int pageSize) =>
    await _db.Orders
             .AsNoTracking()
             .OrderBy(o => o.CreatedAt)
             .Skip((page - 1) * pageSize)
             .Take(pageSize)
             .ToListAsync();
```

---

## 5. Migration Management

### Naming — business intent, not timestamps

```bash
# RIGHT
dotnet ef migrations add AddOrderStatusColumn
dotnet ef migrations add CreateProductTable
dotnet ef migrations add RenameUserEmailToContactEmail

# WRONG
dotnet ef migrations add Migration1
dotnet ef migrations add UpdateDb
```

### One migration per logical change

Each migration maps to one business-level schema change. Do not bundle unrelated changes into a single migration — it makes rollback harder to reason about.

### No auto-migrate — ever

```csharp
// WRONG — never call this anywhere: startup, tests, local dev, production
await context.Database.MigrateAsync();
await context.Database.EnsureCreatedAsync();
```

Auto-migrate is unconditionally banned. Reasons:
- Multiple service instances starting simultaneously cause race conditions
- Cannot be rolled back safely
- Schema changes must be intentional, reviewed operations — not side effects

Apply migrations explicitly via CLI or CI/CD pipeline:

```bash
dotnet ef database update --connection "..." --project Infrastructure
```

### Seed data

Keep seed data separate from migration logic. Use `IEntityTypeConfiguration.HasData()` for static reference data:

```csharp
public class StatusConfiguration : IEntityTypeConfiguration<OrderStatus>
{
    public void Configure(EntityTypeBuilder<OrderStatus> builder)
    {
        builder.HasData(
            new OrderStatus { Id = 1, Name = "Pending" },
            new OrderStatus { Id = 2, Name = "Confirmed" }
        );
    }
}
```

For environment-specific or large seed datasets, use a dedicated seeder class invoked explicitly — not tied to migrations.

---

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `IQueryable` returned from Repository | Materialize with `ToListAsync()` inside Repository |
| `SaveChanges` called inside Repository | Remove — only `IUnitOfWork.CommitAsync()` commits |
| Lazy loading enabled | Disable with `UseLazyLoadingProxies(false)` |
| DbContext shared across threads | Never share — it is not thread-safe; use Scoped lifetime |
| Entity returned directly from Repository | Wrap in Aggregate; never expose raw entities |
| Multiple `Include` on collections without `AsSplitQuery` | Add `AsSplitQuery()` to prevent cartesian explosion |
| Missing `AsNoTracking()` on reads | Add to every query that does not write back |
| `Select` projection done in memory after `ToList()` | Move `Select` into the EF query before materialization |
| `MigrateAsync()` / `EnsureCreatedAsync()` anywhere in code | Remove entirely — use `dotnet ef database update` in deployment pipeline |
| Data Annotations on domain models | Use `IEntityTypeConfiguration<T>` instead |
