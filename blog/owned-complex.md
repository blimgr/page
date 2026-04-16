# Entity Framework Core: Owned Types vs. Complex Types vs. Navigation Properties

When modeling data in EF Core, deciding how to group properties into objects is a critical architectural choice. For years, we relied on **Owned Types**. With EF Core 8, we saw the introduction of **Complex Types**. And of course, there is always the standard **Navigation Property**.

In this post, we’ll break down what each one is, how they differ, and specifically how they handle "sharing" data in your code.

---

## 1. The Three Contenders

### A. Plain Navigation Properties (Entities)
This is the standard "Foreign Key" relationship. The related object is its own entity with its own ID.
* **Database:** Exists in its own table.
* **Identity:** Has a Primary Key.
* **Sharing:** Multiple parents can point to the exact same row in the database.

### B. Owned Types
Owned types allow you to group properties into a class that doesn't have its own primary key. It "borrows" the identity of the parent.
* **Database:** Usually stored in the same table as the parent (Table Splitting).
* **Identity:** Uses a "Shadow Key" linked to the parent.
* **Sharing:** **Cannot be shared** in the database. An instance belongs to exactly one parent.

### C. Complex Types (EF Core 8+)
The new "Value Object" champion. These are groups of properties with no identity whatsoever.
* **Database:** Always stored in the same table as the parent.
* **Identity:** None.
* **Sharing:** Can be "value-shared" in code (see below), but are copied into separate columns in the database.

---

## 2. Feature Comparison Table

| Feature | **Plain Navigation** | **Owned Types** | **Complex Types** |
| :--- | :--- | :--- | :--- |
| **Primary Key** | Yes (Unique ID) | Shadow Key (Owner's ID) | **No Key** |
| **DB Storage** | Separate Table | Same or Separate Table ¹ | **Same Table Only** ² |
| **Nullability** | Can be null | Can be null | Non-null (EF8); Nullable (EF9+) |
| **Collections** | `List<Entity>` | `OwnsMany` | Not supported (EF8) |
| **Instance Sharing** | Allowed | **Forbidden** (Throws error) | **Allowed** (Value-copy) |

> ¹ Owned Types can be split to a separate table via `ToTable()`.
> ² Complex Types do **not** support `ToTable()` and always stay in the parent's table.

---

## 3. The "Assignment" Test: Sharing in Code

This is the most subtle but important difference between Owned and Complex types.

### The Owned Type Problem
Because Owned Types are tracked as entities, they are "jealous." If you try to assign one instance to two parents, EF Core will throw an exception because it sees one "identity" being claimed by two owners.

```csharp
var myAddress = new Address { Street = "123 Main St" };
user1.Address = myAddress;
user2.Address = myAddress; // ERROR: This will fail during SaveChanges!
```

### The Complex Type Solution
Complex Types behave like `int` or `string`. They represent a **value**. EF Core doesn't care about the object reference; it only cares about the data inside.

```csharp
var myPrice = new Money(99.99m, "USD");
product1.Price = myPrice;
product2.Price = myPrice; // Works perfectly!
```
*Note: In the database, the price values will be copied into the columns for both products. They are not "linked," but the code is much cleaner.*

### The Tracking Difference
Owned Types are tracked as full entities — you can inspect or manipulate their change-tracking state via `context.Entry()`. Complex Types have no individual tracking entry; EF Core snapshots their values as part of the parent.

```csharp
// Owned Type — individual entry access works:
var entry = context.Entry(user.HomeAddress);
Console.WriteLine(entry.State); // e.g., Modified

// Complex Type — no individual entry; this throws:
context.Entry(product.Size); // ERROR: InvalidOperationException
```

---

## 4. Code Examples

### Standard Navigation Property
```csharp
public class Blog {
    public int Id { get; set; }
    public Author Author { get; set; } // Independent Entity
}
```

### Owned Type
```csharp
public class User {
    public int Id { get; set; }
    public Address HomeAddress { get; set; }
}

// Configuration
modelBuilder.Entity<User>().OwnsOne(u => u.HomeAddress);
```

### Owned Type with Explicit Key (Separate Table)
```csharp
// Only valid when the owned type lives in its own table
modelBuilder.Entity<User>().OwnsOne(u => u.Address, a => {
    a.HasKey(x => x.Id);
    a.ToTable("Addresses");
});
```

### Owned Collection (`OwnsMany`)
```csharp
public class User {
    public int Id { get; set; }
    public ICollection<PhoneNumber> PhoneNumbers { get; set; }
}

// Configuration
modelBuilder.Entity<User>().OwnsMany(u => u.PhoneNumbers);
```

### Complex Type (Value Object)
```csharp
public class Product {
    public int Id { get; set; }
    public string Name { get; set; }
    public Dimensions Size { get; set; }
}

public record Dimensions(double Height, double Width);

// Configuration
modelBuilder.Entity<Product>().ComplexProperty(p => p.Size);
```

---

## 5. Summary: Which one should I use?

### Use **Plain Navigation Properties** when:
* The object has its own lifecycle (e.g., a `Customer` exists even without an `Order`).
* You need to query the object directly: `_context.Authors.ToList()`.
* The object is truly shared (one `Category` used by many `Products`).

### Use **Owned Types** when:
* The data belongs strictly to one parent.
* You need a **Collection** of child objects (e.g., `User.PhoneNumbers`).
* You need the property to be optional (null).

### Use **Complex Types** when:
* You are following **Domain-Driven Design (DDD)** and using Value Objects.
* You want the performance of a single table but the cleanliness of C# objects.
* You want to reuse the same instance in code across multiple entities without worrying about tracking errors.

---

**Final Tip:** If it's a `record` and it represents a single concept (like `Money`, `Coordinates`, or `Address`), start with a **Complex Type**. If it turns out you need a list of them, move to an **Owned Type**. If you need to search for it independently, make it a **Plain Entity**.