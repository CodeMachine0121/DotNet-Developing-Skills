---
name: dotnet-unit-test
description: Use when writing or reviewing unit tests in .NET Core C# projects. Applies to test structure, naming, mocking, assertions, and framework setup using TUnit, NSubstitute, and FluentAssertions.
---

# .NET Core Unit Testing Style

## Stack

| Role | Library | Version |
|------|---------|---------|
| Test framework | TUnit | latest |
| Mock framework | NSubstitute | latest |
| Assertion library | FluentAssertions | **pinned 7.2.2** |

### Package Reference (csproj)

```xml
<PackageReference Include="TUnit" Version="*" />
<PackageReference Include="NSubstitute" Version="*" />
<PackageReference Include="FluentAssertions" Version="[7.2.2]" />
```

FluentAssertions **must** use bracket notation `[7.2.2]` to pin the exact version.

---

## Rules at a Glance

| Rule | Description |
|------|-------------|
| One assertion per test | Each test case contains exactly one `Should()` call |
| Naming: `Method_Scenario_ExpectedBehavior` | Test name is the specification |
| AAA structure | Arrange / Act / Assert separated by blank lines |
| Mock only external dependencies | Repositories, HTTP clients — not domain objects |
| Test public API only | Never test private or internal implementation details |
| Fluent test body | Readable as a sentence; no procedural clutter |
| `Given...` prefix | Methods that set up mock return values |
| `Create...` prefix | Methods that instantiate objects under test or test data |
| Setup in `[Before(Test)]` | SUT and injected interfaces declared and initialized there |

---

## Test Structure

### Naming Convention

```
MethodName_Scenario_ExpectedBehavior
```

```csharp
// examples
PlaceOrder_WithValidDto_ReturnsSuccessResult
PlaceOrder_WhenUserNotFound_ThrowsNotFoundException
GetTotal_WithDiscount_AppliesDiscountRate
```

### AAA Layout

```csharp
[Test]
public async Task PlaceOrder_WithValidDto_ReturnsSuccessResult()
{
    // Arrange
    var dto = CreateValidOrderDto();
    GivenUserExists(dto.UserId);

    // Act
    var result = await _sut.PlaceOrder(dto);

    // Assert
    result.IsSuccess.Should().BeTrue();
}
```

---

## Setup

Declare the SUT and all substituted interfaces as fields. Initialize them in `[Before(Test)]`.

```csharp
public class OrderServiceTests
{
    private OrderService _sut;
    private IOrderRepository _orderRepo;
    private IUserRepository _userRepo;

    [Before(Test)]
    public void Setup()
    {
        _orderRepo = Substitute.For<IOrderRepository>();
        _userRepo = Substitute.For<IUserRepository>();
        _sut = new OrderService(_orderRepo, _userRepo);
    }
}
```

---

## Helper Methods

### `Given...` — mock return values

Extract mock setup into `Given` methods when it would clutter the Arrange block.

```csharp
private void GivenUserExists(Guid userId)
{
    _userRepo.Get(userId).Returns(new User { Id = userId, Name = "Alice" });
}

private void GivenUserNotFound(Guid userId)
{
    _userRepo.Get(userId).Returns((User?)null);
}
```

### `Create...` — object construction

Extract `new` expressions for the SUT's inputs into `Create` methods.

```csharp
private static CreateOrderDto CreateValidOrderDto() =>
    new(UserId: Guid.NewGuid(), ProductIds: [Guid.NewGuid()], CouponCode: null);

private static CreateOrderDto CreateOrderDtoWithCoupon(string code) =>
    new(UserId: Guid.NewGuid(), ProductIds: [Guid.NewGuid()], CouponCode: code);
```

---

## Assertions

Use FluentAssertions. One `Should()` chain per test.

```csharp
// value
result.Should().Be(42);
result.Should().BeNull();
result.IsSuccess.Should().BeTrue();

// collection
items.Should().HaveCount(3);
items.Should().Contain(x => x.Id == expectedId);

// async exception
await _sut.Invoking(s => s.PlaceOrder(dto))
    .Should().ThrowAsync<NotFoundException>();
```

---

## What to Mock

| Mock it | Don't mock it |
|---------|--------------|
| Repository interfaces | Domain models (`Order`, `User`) |
| HTTP clients / external APIs | Value objects |
| Message bus / email sender | Pure computation services |
| File system abstractions | In-memory collections |

---

## Full Example

```csharp
public class OrderServiceTests
{
    private OrderService _sut;
    private IOrderRepository _orderRepo;
    private IUserRepository _userRepo;

    [Before(Test)]
    public void Setup()
    {
        _orderRepo = Substitute.For<IOrderRepository>();
        _userRepo = Substitute.For<IUserRepository>();
        _sut = new OrderService(_orderRepo, _userRepo);
    }

    [Test]
    public async Task PlaceOrder_WithValidDto_SavesOrder()
    {
        // Arrange
        var dto = CreateValidOrderDto();
        GivenUserExists(dto.UserId);

        // Act
        await _sut.PlaceOrder(dto);

        // Assert
        await _orderRepo.Received(1).Save(Arg.Any<Order>());
    }

    [Test]
    public async Task PlaceOrder_WithValidDto_ReturnsSuccessResult()
    {
        // Arrange
        var dto = CreateValidOrderDto();
        GivenUserExists(dto.UserId);

        // Act
        var result = await _sut.PlaceOrder(dto);

        // Assert
        result.IsSuccess.Should().BeTrue();
    }

    [Test]
    public async Task PlaceOrder_WhenUserNotFound_ThrowsNotFoundException()
    {
        // Arrange
        var dto = CreateValidOrderDto();
        GivenUserNotFound(dto.UserId);

        // Act & Assert
        await _sut.Invoking(s => s.PlaceOrder(dto))
            .Should().ThrowAsync<NotFoundException>();
    }

    // --- Given ---

    private void GivenUserExists(Guid userId) =>
        _userRepo.Get(userId).Returns(new User { Id = userId });

    private void GivenUserNotFound(Guid userId) =>
        _userRepo.Get(userId).Returns((User?)null);

    // --- Create ---

    private static CreateOrderDto CreateValidOrderDto() =>
        new(UserId: Guid.NewGuid(), ProductIds: [Guid.NewGuid()], CouponCode: null);
}
```

---

## Violation Quick Reference

| Symptom | Fix |
|---------|-----|
| Multiple `Should()` calls in one test | Split into separate test methods |
| Mock setup inline in test body | Extract to `Given...` method |
| `new SomeDto(...)` repeated across tests | Extract to `Create...` method |
| `private` fields initialized inline | Move to `[Before(Test)]` |
| Test name is `Test1` or `ShouldWork` | Rename to `Method_Scenario_ExpectedBehavior` |
| Mocking a domain model or value object | Use a real instance instead |
| Testing via reflection or internal members | Redesign — test the public API |
| Missing blank lines between AAA blocks | Add blank lines to separate Arrange / Act / Assert |
