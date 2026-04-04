# The Dependency Inversion Principle

The Dependency Inversion Principle is one of the five SOLID principles introduced by Robert C. Martin (known as Uncle Bob) — 
a set of guidelines that form a foundation for writing object-oriented code that is maintainable, 
flexible, and easy to change. SOLID stands for Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion. 
Each principle addresses a specific way that code tends to go wrong as systems grow.

## The Definition

Robert Martin states the Dependency Inversion Principle in two parts:

> **A.** High-level modules should not depend on low-level modules. Both should depend on abstractions.
>
> **B.** Abstractions should not depend on details. Details should depend on abstractions.


## The Scenario

In the following sections we will attempt to explain the Dependency Inversion Principle. To clearly illustrate these concepts, we will use the following scenario through the rest of the post.

We have an `OrderService` — a class responsible for placing customer orders. It is a **high-level module**: it contains the important decisions in our system, the business rules, the orchestration logic, the policy. It answers the question *why*. It expresses what our system is *for*.

To do its job, it needs to persist orders somewhere, so it works with a `MySQLDatabase`. That is a **low-level module**: it handles the implementation details, the *how*. It doesn't know anything about the business — it just knows how to store data in a specific technical way.

The question the Dependency Inversion Principle answers is: how should these two modules relate to each other?

## Part A: High-level modules should not depend on low-level modules. Both should depend on abstractions

Part A is about the relationship between these two kinds of modules — and as we saw in the scenario, `OrderService` is the high-level one and `MySQLDatabase` is the low-level one.

The natural, instinctive way to write code is to have the high-level module directly create and call the low-level one:

```csharp
public class OrderService
{
    private readonly MySQLDatabase _db;

    public OrderService()
    {
        _db = new MySQLDatabase();
    }

    public void PlaceOrder(Order order)
    {
        _db.Save(order);
    }
}
```

This seems straightforward, but it creates a problem: `OrderService` is now tightly coupled to `MySQLDatabase`. If you want to swap the database, write a test with a fake database, or reuse `OrderService` in a different context, you have to reach inside your business logic to do it.

Part A says this is wrong. Instead, both modules should depend on an **abstraction** — an interface or abstract class that defines a contract without specifying the implementation:

```csharp
public interface IDatabase
{
    void Save(Order order);
}

public class OrderService
{
    private readonly IDatabase _db;

    public OrderService(IDatabase db)
    {
        _db = db;
    }

    public void PlaceOrder(Order order)
    {
        _db.Save(order);
    }
}

public class MySQLDatabase : IDatabase
{
    public void Save(Order order)
    {
        // MySQL-specific implementation
    }
}
```

Now `OrderService` knows nothing about MySQL. It only knows about `IDatabase`. You can swap, mock, or extend the implementation without touching the business logic.

### What Is Actually Being Inverted?

Looking at the two versions of the code, something has changed beyond just introducing an interface — the direction of the dependency itself has flipped.

In the naive version, the dependency flows top-down:

```
OrderService  ──depends on──►  MySQLDatabase
```

`OrderService` knows about `MySQLDatabase`. If `MySQLDatabase` changes, `OrderService` may have to change too. The high-level policy is at the mercy of the low-level detail.

After applying Part A, the dependency arrows change:

```
OrderService  ──depends on──►  IDatabase  ◄──implements──  MySQLDatabase
```

`MySQLDatabase` now depends on `IDatabase` — it has to conform to a contract it did not define. The dependency arrow from the low-level module has been inverted. Instead of the high-level module reaching down to grab the low-level one, the low-level one reaches up to meet a standard set for it. That is the first inversion: **the direction of the dependency**.

There is a second inversion too: in a naive design, the interface tends to be defined in terms of what the low-level module offers. After applying DIP, it is defined in terms of what the high-level module needs. The low-level module has to conform to that contract, not the other way around.


## Part B: Abstractions should not depend on details. Details should depend on abstractions

Part A tells you that both sides should depend on an abstraction. Part B tells you how that abstraction should be designed.

The rule is: the abstraction should not be shaped by the implementation behind it. It should reflect the needs of the high-level module, not the capabilities of the low-level one.

When people think about an abstraction depending on a detail, they usually think of type dependencies — an interface that references a concrete type like `MySqlConnection` in its signature. But a dependency on a detail can leak through in subtler ways too:

- **Naming** — a method called `ExecuteSqlQuery()` on an `IDatabase` interface encodes a detail in the name itself, even if the signature uses only primitive types.
- **Parameter shape** — a method that takes a `connectionString` implies a connection-string-based system. A method that takes a `SqlTransaction` implies SQL. The parameters may be common types, but their shape reveals the underlying detail.
- **Return values** — returning a `DataTable` or `SqlDataReader` ties callers to a relational concept, even if the interface otherwise looks abstract.
- **Exception contracts** — if the interface's implied contract includes throwing `SqlException`, it has leaked a detail through its error model.
- **Method structure** — an interface with `BeginTransaction()`, `Commit()`, and `Rollback()` is encoding a transactional relational model, even if every type in the signatures is generic.

Any of these can make an abstraction secretly dependent on the detail it is supposed to hide.

### Context determines what counts as a detail

It is important to note that "detail" is not absolute — it is relative to the context the abstraction lives in and the variation it is meant to accommodate.

Consider an `IDatabase` interface that exposes `ExecuteSqlQuery()` and `BeginTransaction()`. In a system where the database layer could be a document store, a key-value cache, or a relational database, these are details leaking upward — the abstraction is assuming an RDBMS when it is supposed to be neutral. But in a system where the database is always relational — where the only question is *which* RDBMS — those concepts are not details anymore. They are legitimate parts of the contract. An interface shaped around RDBMS concepts is entirely appropriate when RDBMS is a given.

The question to ask is: *what variation is this abstraction meant to hide?* A detail is anything that assumes away a variation the abstraction is supposed to leave open.

## When A Holds but B Does Not

Using our scenario, `OrderService` depends on `IDatabase` — not on `MySQLDatabase` directly. Part A is satisfied. But look at how `IDatabase` is defined:

```csharp
public interface IDatabase
{
    MySqlConnection GetConnection();
    void Save(Order order);
}

public class OrderService
{
    private readonly IDatabase _db;

    public OrderService(IDatabase db)
    {
        _db = db;
    }

    public void PlaceOrder(Order order)
    {
        _db.Save(order);
    }
}

public class MySQLDatabase : IDatabase
{
    public MySqlConnection GetConnection()
    {
        // returns a MySQL connection
    }

    public void Save(Order order)
    {
        // saves using MySQL
    }
}
```

Both sides depend on the abstraction and the dependency arrows are pointing in the right direction — Part A holds.

But Part B is violated: `IDatabase` exposes `GetConnection()` which returns a `MySqlConnection` — a MySQL-specific type. The abstraction has been shaped by the implementation behind it. If you tried to introduce a `PostgreSQLDatabase`, it could not honestly implement `GetConnection()`. The interface is not truly neutral — it is secretly tied to MySQL through its own signature. The abstraction depends on the detail, which is exactly what Part B forbids.

## When B Holds but A Does Not

Now `IDatabase` is clean and neutral — Part B is satisfied. But look at how `OrderService` uses it:

```csharp
public interface IDatabase
{
    void Save(Order order);
}

public class MySQLDatabase : IDatabase
{
    public void Save(Order order)
    {
        // saves using MySQL
    }
}

public class OrderService
{
    private readonly MySQLDatabase _db;

    public OrderService()
    {
        _db = new MySQLDatabase();
    }

    public void PlaceOrder(Order order)
    {
        _db.Save(order);
    }
}
```

`IDatabase` is a clean, neutral abstraction that leaks no implementation details and `MySQLDatabase` conforms to it properly — Part B holds.

But Part A is violated: `OrderService` ignores the abstraction entirely and depends directly on `MySQLDatabase`. It instantiates it internally and holds a concrete reference to it. Despite having a perfectly good interface available, the high-level module has reached past it and grabbed the low-level detail directly. Swapping the database still requires changing `OrderService`.

Both parts need to hold for the principle to be satisfied. A clean abstraction means nothing if the high-level module doesn't use it. And depending on an abstraction means nothing if the abstraction is secretly shaped by the detail it hides.
