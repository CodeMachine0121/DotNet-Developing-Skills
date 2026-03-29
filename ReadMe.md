# Coding Skills

A collection of skills covering .NET Core best practices and general engineering methodology.

## Installation

Install from GitHub in Claude Code (two steps):

```
/plugin marketplace add https://github.com/CodeMachine0121/DotNet-Developing-Skills
/plugin install coding-skills@CodeMachine0121-DotNet-Developing-Skills
```

## Usage

Each skill is independently invocable via the `Skill` tool. Claude will automatically trigger the relevant skill based on context — for example, when refactoring .NET code, `dotnet-refactor` activates; when writing tests, `dotnet-unit-test` activates.

You can also invoke a skill explicitly by name:

| Skill | When to use |
|-------|-------------|
| `dotnet-refactor` | Refactoring C# code or reviewing design against James's style |
| `dotnet-architecture` | Designing or reviewing .NET Core project layer structure |
| `dotnet-efcore` | Implementing or reviewing EF Core usage |
| `dotnet-unit-test` | Writing or reviewing unit tests in .NET Core |
| `tidy` | About to change code and the structure feels messy |

## Skills

### [`tidy`](skills/tidy/SKILL.md)

Applies Kent Beck's *Tidy First?* methodology — separate structural changes from behavioral changes and sequence them deliberately.

| Rule | Summary |
|------|---------|
| Never mix tidying and feature work | Structural and behavioral changes must be in separate commits |
| Tidy first / after / not at all | Decide based on whether the change will be clearer or safer after tidying |
| 15 tidyings | Guard Clauses, Explaining Variables, Extract Helper, Dead Code, Cohesion Order, and more |
| One tidy per commit | Small, reversible, reviewable in minutes |
| Stop on bug discovery | File the bug — don't fix mid-tidy |

---

### [`dotnet-refactor`](skills/dotnet-refactor/SKILL.md)

Refactoring rules for .NET Core C# projects following James's style.

| Rule | Summary |
|------|---------|
| No private methods | Extract a class or inline the expression |
| No private static methods | Move to the type that owns the data |
| Method ≤ 5 responsibilities | Split or extract a Decorator |
| Parameters > 2 | Extract into a DTO record |
| Complex pre-conditions | Decorator pattern |
| Conditional branch implementations | Strategy pattern |
| Pure service / domain layer | No framework imports — depend only on interfaces |
| Business logic | Fluent method chaining |
| Variable assignment | Declare and assign in one statement |
| Model conversions | `To...()` instance methods on the source model |
| DI lifetimes | Never inject Scoped/Transient into a Singleton |

---

### [`dotnet-architecture`](skills/dotnet-architecture/SKILL.md)

Layered architecture for .NET Core projects.

```
Controller
    ↓
Application Service
    ↓
Domain Service  ←→  Repository / Proxy
```

| Layer | Responsibility |
|-------|---------------|
| Controller | Parse HTTP request → assemble input DTO → map result to ViewModel |
| Application Service | Orchestration only — no business rules |
| Domain Service | Business rules — depends on Repository/Proxy interfaces |
| Repository | Persistence abstraction — returns Aggregates |
| Proxy | External API abstraction — returns DTOs |

Cross-cutting concerns: **Action Filters** for API pre-conditions, **Middleware** for global pipeline behavior (exception handling, logging, CORS).

---

### [`dotnet-efcore`](skills/dotnet-efcore/SKILL.md)

EF Core best practices.

| Rule | Summary |
|------|---------|
| EF only in Repository | No EF references in Service or Domain layers |
| Scoped DbContext | Always register as `Scoped` |
| Disable lazy loading | `UseLazyLoadingProxies(false)` |
| `AsNoTracking()` on reads | All read-only queries must opt out of change tracking |
| No `IQueryable` outside Repository | Never leak query composition across layer boundaries |
| `SaveChanges` via Unit of Work only | Repositories stage — `IUnitOfWork.CommitAsync()` commits |
| One migration per logical change | Named after business intent, not timestamps |
| No auto-migrate — ever | `MigrateAsync()` and `EnsureCreatedAsync()` are banned |

---

### [`dotnet-unit-test`](skills/dotnet-unit-test/SKILL.md)

Unit testing style for .NET Core projects.

**Stack:** TUnit + NSubstitute + FluentAssertions (pinned 7.2.2)

| Rule | Summary |
|------|---------|
| One assertion per test | Exactly one `Should()` call per test method |
| Naming: `Method_Scenario_ExpectedBehavior` | Test name is the specification |
| AAA structure | Arrange / Act / Assert separated by blank lines |
| Mock only external dependencies | Repositories, HTTP clients — not domain objects |
| Test public API only | Never test private or internal members |
| `Given...` prefix | Methods that set up mock return values |
| `Create...` prefix | Methods that instantiate the SUT or test data |
| Setup in `[Before(Test)]` | SUT and interfaces declared and initialized there |
