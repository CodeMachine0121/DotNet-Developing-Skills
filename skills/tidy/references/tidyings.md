# The 15 Tidyings — Reference

Each tidying is a pure structural change: behavior stays identical before and after.

---

## 1. Guard Clauses

Move preconditions to the top and return early. Reduces nesting.

**Before:**
```csharp
void Process(Order order) {
    if (order != null) {
        if (order.IsValid) {
            // main logic...
        }
    }
}
```

**After:**
```csharp
void Process(Order order) {
    if (order == null) return;
    if (!order.IsValid) return;
    // main logic...
}
```

---

## 2. Dead Code

Delete code that is never executed. Don't comment it out — version control preserves history.

```csharp
// Delete: unused methods, unreachable branches, disabled feature flags, commented-out blocks
```

---

## 3. Normalize Symmetries

When two pieces of code do similar things, make them look identical so the pattern is obvious.

```csharp
// Before: inconsistent null checks across methods
// After: every method checks null the same way
```

---

## 4. New Interface, Old Implementation

Create a new clean signature that wraps the old messy one. Callers migrate gradually.

```csharp
// Old
void SendEmail(string to, string subject, string body, bool isHtml, string replyTo) { ... }

// New (wraps old)
void SendEmail(EmailMessage message) => SendEmail(message.To, message.Subject, ...);
```

---

## 5. Reading Order

Arrange code so a reader encounters definitions before usages. Top-to-bottom narrative flow.

- Move helper methods below the methods that call them (or above, per language convention)
- Keep the "story" of what a class does visible at the top

---

## 6. Cohesion Order

Move code that changes together, next to each other. If two methods always change as a pair, place them adjacent.

---

## 7. Move Declaration And Initialization Together

```csharp
// Before
int count;
DoSomethingElse();
count = GetCount();

// After
DoSomethingElse();
int count = GetCount();
```

---

## 8. Explaining Variables

Extract a sub-expression into a named variable to document intent.

```csharp
// Before
if (user.Age >= 18 && user.Country == "TW" && !user.IsBanned) { ... }

// After
bool isEligible = user.Age >= 18 && user.Country == "TW" && !user.IsBanned;
if (isEligible) { ... }
```

---

## 9. Explaining Constants

Replace magic values with named constants.

```csharp
// Before
if (retries > 3) throw ...;

// After
const int MaxRetries = 3;
if (retries > MaxRetries) throw ...;
```

---

## 10. Explicit Parameters

Remove hidden state (globals, thread-locals, ambient context) and pass values explicitly.

```csharp
// Before: reads CurrentUser from static context
void Save() { var user = CurrentContext.User; ... }

// After
void Save(User user) { ... }
```

---

## 11. Chunk Statements

Add blank lines between logical phases of a method. No code changes, just whitespace.

```csharp
void Process(Order order) {
    // validate
    if (!order.IsValid) throw ...;

    // calculate
    var total = order.Lines.Sum(l => l.Total);

    // persist
    _repo.Save(order);
}
```

---

## 12. Extract Helper

Pull a cohesive chunk of logic into a named method. Name it for what it does, not how.

```csharp
// Before: inline validation logic
// After
bool IsEligibleForDiscount(Order order) => order.Total > 1000 && order.IsFirstPurchase;
```

---

## 13. One Pile

When code is over-split and hard to understand, temporarily consolidate it into one place. Then re-split with better boundaries.

Use when: code is spread across 5 tiny methods and you can't see the whole picture.

---

## 14. Explaining Comments

Add a comment that answers *why* — the intent, constraint, or business rule that isn't obvious from the code.

```csharp
// Taiwan law requires a 7-day cooling-off period for online purchases
if (daysSincePurchase < 7) AllowReturn(order);
```

---

## 15. Delete Redundant Comments

Remove comments that just restate what the code already clearly says.

```csharp
// Before
// increment count by 1
count++;

// After
count++;
```
