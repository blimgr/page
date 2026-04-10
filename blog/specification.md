# The Specification Pattern: Writing Business Rules That Don't Rot

*A deep dive into one of the most underrated patterns in software design*

---

There's a special kind of frustration that comes from hunting down a business rule scattered across a dozen `if` statements. You find one copy in the service layer, another in a validator, maybe a third lurking inside a LINQ query, and a fourth that someone baked into a database stored procedure three years ago. They're all *almost* the same. Almost.

The **Specification Pattern** exists precisely to end this suffering.

---

## What Is the Specification Pattern?

The Specification Pattern is a behavioral design pattern that lets you encapsulate a business rule — or any boolean predicate — into a standalone, reusable object. Instead of embedding "is this customer eligible for a discount?" logic directly into your service or repository, you package it into a `Specification` class that can be named, combined, tested, and reused.

Eric Evans and Martin Fowler first described it formally in their 2000 paper *"Specifications"*, and it has since become a cornerstone of Domain-Driven Design (DDD).

The core idea is elegantly simple:

> A specification is an object that answers **"does this thing satisfy this rule?"**

---

## The Problem It Solves

Consider an e-commerce application. You have a rule: **customers who have placed at least 5 orders and have no outstanding payments are eligible for a loyalty discount**.

Without the pattern, this might look like:

```csharp
// In OrderService
if (customer.Orders.Count >= 5 && !customer.HasOutstandingPayments)
{
    ApplyDiscount(order);
}

// In CustomerReportService
var eligibleCustomers = customers
    .Where(c => c.Orders.Count >= 5 && !c.HasOutstandingPayments)
    .ToList();

// In a validator somewhere
if (!(customer.Orders.Count >= 5 && !customer.HasOutstandingPayments))
{
    throw new ValidationException("Customer not eligible.");
}
```

This is fragile. When the business changes the threshold from 5 to 10 orders, you hunt down every occurrence. You miss one. A bug is born.

---

## The Pattern in Practice

### Step 1: Define the Interface

Start with a simple contract:

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
}
```

### Step 2: Implement a Concrete Specification

```csharp
public class LoyaltyDiscountEligibleSpecification : ISpecification<Customer>
{
    private const int MinimumOrders = 5;

    public bool IsSatisfiedBy(Customer customer)
    {
        return customer.Orders.Count >= MinimumOrders
            && !customer.HasOutstandingPayments;
    }
}
```

Now the rule has a *name*, a *home*, and a single place to update.

### Step 3: Use It

```csharp
var spec = new LoyaltyDiscountEligibleSpecification();

if (spec.IsSatisfiedBy(customer))
{
    ApplyDiscount(order);
}
```

---

## The Real Power: Composability

Where the pattern truly shines is in **combining specifications** using logical operators — AND, OR, and NOT.

### Composite Specifications

```csharp
public class AndSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _left;
    private readonly ISpecification<T> _right;

    public AndSpecification(ISpecification<T> left, ISpecification<T> right)
    {
        _left = left;
        _right = right;
    }

    public bool IsSatisfiedBy(T entity)
        => _left.IsSatisfiedBy(entity) && _right.IsSatisfiedBy(entity);
}

public class NotSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _inner;

    public NotSpecification(ISpecification<T> inner) => _inner = inner;

    public bool IsSatisfiedBy(T entity) => !_inner.IsSatisfiedBy(entity);
}
```

You can add fluent helpers directly to the base class:

```csharp
public abstract class Specification<T> : ISpecification<T>
{
    public abstract bool IsSatisfiedBy(T entity);

    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);

    public Specification<T> Not()
        => new NotSpecification<T>(this);
}
```

Now your business rules read almost like prose:

```csharp
var eligibleForCampaign = new LoyaltyDiscountEligibleSpecification()
    .And(new ActiveCustomerSpecification())
    .And(new NotSpecification<Customer>(new BlacklistedCustomerSpecification()));

var qualifyingCustomers = customers.Where(eligibleForCampaign.IsSatisfiedBy);
```

---

## Specifications and Persistence

One common challenge: you need the same rule to work **in memory** (for validation) *and* **at the database level** (for efficient querying). Evaluating a specification in memory on 2 million records is not a strategy.

The solution is to extend your specification to return an **expression tree**:

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    Expression<Func<T, bool>> ToExpression();
}
```

```csharp
public class LoyaltyDiscountEligibleSpecification : ISpecification<Customer>
{
    public Expression<Func<Customer, bool>> ToExpression()
    {
        return customer => customer.Orders.Count >= 5
                        && !customer.HasOutstandingPayments;
    }

    public bool IsSatisfiedBy(Customer customer)
        => ToExpression().Compile()(customer);
}
```

Your repository can then pass the expression straight to Entity Framework:

```csharp
public IEnumerable<Customer> FindSatisfying(ISpecification<Customer> spec)
{
    return _dbContext.Customers
        .Where(spec.ToExpression())
        .ToList();
}
```

The same specification powers both your in-memory checks and your database queries. One source of truth.

---

## When to Use It

The Specification Pattern earns its keep when:

- **Business rules are complex and likely to change.** Simple `if` checks don't need a pattern. Nuanced, policy-driven rules do.
- **The same rule appears in multiple places.** Filtering, validation, and querying all need the same criteria.
- **Rules need to be composed dynamically.** Building a search with optional filters is a natural fit.
- **You practice DDD.** Specifications are a first-class citizen in the domain model.

## When to Skip It

It's worth being honest about overhead. The pattern adds indirection and abstraction. Avoid it when:

- The rule is trivial and used in only one place.
- Your team finds the extra classes confusing without clear benefit.
- The domain is anemic and business logic lives in a script, not a rich model.

---

## A Complete Example: Product Search

Here's a realistic, self-contained example tying it all together:

```csharp
// Specifications
public class InStockSpecification : Specification<Product>
{
    public override bool IsSatisfiedBy(Product p) => p.Stock > 0;
}

public class PriceRangeSpecification : Specification<Product>
{
    private readonly decimal _min;
    private readonly decimal _max;

    public PriceRangeSpecification(decimal min, decimal max)
    {
        _min = min;
        _max = max;
    }

    public override bool IsSatisfiedBy(Product p)
        => p.Price >= _min && p.Price <= _max;
}

public class CategorySpecification : Specification<Product>
{
    private readonly string _category;

    public CategorySpecification(string category) => _category = category;

    public override bool IsSatisfiedBy(Product p)
        => p.Category.Equals(_category, StringComparison.OrdinalIgnoreCase);
}

// Usage
var searchSpec = new InStockSpecification()
    .And(new PriceRangeSpecification(10m, 100m))
    .And(new CategorySpecification("Electronics"));

var results = productRepository.FindSatisfying(searchSpec);
```

Adding a new filter — say, "on sale" — is a new class, not a modification of existing logic. Open/Closed Principle in action.

---

## Don't Roll Your Own: Ardalis.Specification

Building the `AndSpecification`, `OrSpecification`, expression tree plumbing, and repository integration yourself is instructive — but in a real project you probably shouldn't. **[Ardalis.Specification](https://www.nuget.org/packages/Ardalis.Specification)** is a battle-tested .NET library by Steve "ardalis" Smith that gives you all of this out of the box.

It's used inside Microsoft's own reference application **eShopOnWeb** and the Clean Architecture solution template, and has accumulated over **15 million NuGet downloads**. Version 9 (the current major release) is maintained primarily by Fati Iseni and targets .NET 8+.

### Installation

```bash
dotnet add package Ardalis.Specification
dotnet add package Ardalis.Specification.EntityFrameworkCore  # if using EF Core
```

### What It Gives You

Instead of writing your own base class and composite specifications, you inherit from `Specification<T>` and use its built-in fluent builder:

```csharp
using Ardalis.Specification;

public class ActiveCustomersWithOrdersSpec : Specification<Customer>
{
    public ActiveCustomersWithOrdersSpec(int minimumOrders)
    {
        Query
            .Where(c => c.IsActive && c.Orders.Count >= minimumOrders)
            .Include(c => c.Orders)
            .OrderBy(c => c.LastName);
    }
}
```

Notice that `Where`, `Include`, `OrderBy`, and pagination are all first-class citizens on the fluent `Query` builder — no more LINQ scattered through your repositories.

### Repository Integration

The library ships with a `RepositoryBase<T>` for EF Core that understands specifications natively:

```csharp
// Your repository inherits from RepositoryBase<T>
public class CustomerRepository : RepositoryBase<Customer>, ICustomerRepository
{
    public CustomerRepository(AppDbContext dbContext) : base(dbContext) { }
}

// Usage in a service
var spec = new ActiveCustomersWithOrdersSpec(minimumOrders: 5);
var customers = await _customerRepository.ListAsync(spec);
```

No custom `FindSatisfying` method needed. The evaluator translates the specification into an EF Core query automatically — `Where` clauses become SQL `WHERE`, `Include` becomes a JOIN, `OrderBy` becomes `ORDER BY`.

### In-Memory Evaluation

For unit testing or in-memory validation, the library also ships with an `InMemorySpecificationEvaluator`:

```csharp
var evaluator = new InMemorySpecificationEvaluator();
var results = evaluator.Evaluate(customersList, new ActiveCustomersWithOrdersSpec(5));
```

The same specification works against both the database and an in-memory list — exactly the dual-mode behavior we built manually in the earlier sections, but without the boilerplate.

### When to Use the Library vs. Roll Your Own

Use **Ardalis.Specification** when you're building a real application and want proven infrastructure. Roll your own when you're learning the pattern, working in a non-.NET stack, or have highly unusual evaluation requirements the library doesn't support.

---

## Summary

| Concern | Without Specification | With Specification |
|---|---|---|
| Business rule location | Scattered | Encapsulated |
| Reuse | Copy-paste | Reference the class |
| Testability | Implicit | Explicit, isolated unit tests |
| Composability | Manual boolean logic | `.And()`, `.Or()`, `.Not()` |
| DB + in-memory parity | Duplicate logic | Single `Expression<T>` |

The Specification Pattern won't solve every design problem, but when business rules are complex, shared, and evolving — which in any meaningful domain they always are — it brings a clarity and resilience that scattered conditionals simply cannot match.

Your future self, staring at a changed requirement at 11pm, will thank you.

---

*Further reading: Martin Fowler's [Specification](https://www.martinfowler.com/apsupp/spec.pdf) paper · Eric Evans' "Domain-Driven Design: Tackling Complexity in the Heart of Software" · [Ardalis.Specification docs](https://specification.ardalis.com/) · [GitHub: ardalis/Specification](https://github.com/ardalis/Specification)*
