# Good Unit Tests

---

## 1. Single Responsibility

Just like the **S** in SOLID — where a class should have only one reason to change — a test should verify only one logical concept. If a test fails, the cause should be immediately obvious — not buried in a chain of unrelated checks.

One logical concept does not strictly mean one `Assert` — sometimes a single concept requires checking multiple related properties (e.g. all fields of a returned object). But in practice, one concept usually maps to one assertion. If you find yourself asserting two unrelated things, that's a sign to split the test.

```csharp
// bad — two unrelated concepts in one test
[Fact]
public void CreateUser_ValidInput_SetsNameAndAdminRole()
{
    var user = new User("Ana", isAdmin: true);
    Assert.Equal("Ana", user.Name);      // concept 1
    Assert.True(user.IsAdmin);           // concept 2 — unrelated, split this out
}

// acceptable — multiple assertions, one concept (the full address is one thing)
[Fact]
public void CreateAddress_ValidInput_SetsAllAddressFields()
{
    var address = new Address("Baker St", "London", "NW1 6XE");
    Assert.Equal("Baker St", address.Street);
    Assert.Equal("London", address.City);
    Assert.Equal("NW1 6XE", address.PostCode);
}

// good — each concept in its own test
[Fact]
public void CreateUser_ValidName_SetsName() { ... }

[Fact]
public void CreateUser_AdminFlag_GrantsAdminRole() { ... }
```

---

## 2. Descriptive Test Names

The name is the first line of your error message. Follow the convention `TestedMethod_ScenarioTested_ExpectedBehaviour` so a failing test tells you *what* broke and *under what condition* — without reading the body.

```csharp
// bad
[Fact]
public void Test1() { ... }

[Fact]
public void Works() { ... }

// good — TestedMethod_ScenarioTested_ExpectedBehaviour
[Fact]
public void GetUser_UserHasNoEmail_ReturnsNull() { ... }

[Fact]
public void PlaceOrder_InsufficientStock_ThrowsInvalidOperationException() { ... }
```

---

## 3. Isolated

No network calls, no disk I/O, no shared global state. Mock or stub external dependencies so the test fails only because of *your code*, not a flaky API or database.

```csharp
// inject deps — easy to swap in tests
public class UserService
{
    private readonly IUserRepository _repo;
    public UserService(IUserRepository repo) => _repo = repo;
}

// in tests — use a mock, not the real database
[Fact]
public void GetUser_ExistingId_ReturnsUser()
{
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.FindById(1)).Returns(new User { Id = 1 });

    var sut = new UserService(mockRepo.Object);
    var result = sut.GetUser(1);

    Assert.NotNull(result);
}
```

---

## 4. Fast

Unit tests should finish in milliseconds. Slow tests kill the feedback loop. If a test needs a real database or real clock, it's an integration test — keep them separate.

```csharp
// bad — real delay in the test
[Fact]
public async Task ExpireToken_AfterTimeout_IsExpired()
{
    var token = new Token(expiresIn: TimeSpan.FromSeconds(5));
    await Task.Delay(5000); // never do this
    Assert.True(token.IsExpired);
}

// good — inject the clock, control time
[Fact]
public void ExpireToken_PastExpiry_IsExpired()
{
    var fakeClock = new FakeClock(DateTime.UtcNow.AddSeconds(-1));
    var token = new Token(expiresAt: DateTime.UtcNow, clock: fakeClock);

    Assert.True(token.IsExpired);
}
```

---

## 5. Tests Public Interface

Test *what* the code does, not *how* it does it internally. Coupling tests to private methods makes refactoring painful without changing any observable behaviour.

```csharp
// bad — tests internal state via reflection
[Fact]
public void AddItem_ValidItem_IncreasesInternalList()
{
    var cart = new Cart();
    cart.Add(new Item(price: 10m));

    var items = (List<Item>)typeof(Cart)
        .GetField("_items", BindingFlags.NonPublic | BindingFlags.Instance)
        .GetValue(cart);

    Assert.Equal(1, items.Count);
}

// good — tests the public contract
[Fact]
public void GetTotal_OneItemAdded_ReturnsTotalPrice()
{
    var cart = new Cart();
    cart.Add(new Item(price: 10m));

    Assert.Equal(10m, cart.GetTotal());
}
```

---

## 6. Deterministic

Same code, same result — always. No dependency on run order, wall-clock time, random seeds, or environment variables left to chance. Flaky tests erode trust in the entire suite.

```csharp
// bad — depends on real wall-clock time
[Fact]
public void GenerateToken_NewToken_IsNotExpired()
{
    var token = new Token(expiresAt: DateTime.UtcNow.AddMinutes(5));
    Assert.False(token.IsExpired); // may fail near midnight, under load, etc.
}

// good — fixed, controlled point in time
[Fact]
public void GenerateToken_ExpiryInFuture_IsNotExpired()
{
    var fixedNow = new DateTime(2024, 1, 1, 12, 0, 0, DateTimeKind.Utc);
    var fakeClock = new FakeClock(fixedNow);
    var token = new Token(expiresAt: fixedNow.AddMinutes(5), clock: fakeClock);

    Assert.False(token.IsExpired);
}
```

---

## 7. Covers Edge Cases

Happy path only is a false sense of security. Test null inputs, empty collections, boundary values, error conditions, and the paths users never intend to take.

```csharp
[Fact]
public void Parse_EmptyString_ReturnsNull() { ... }

[Fact]
public void Parse_ValidInput_ReturnsResult() { ... }

[Fact]
public void Parse_NullInput_ThrowsArgumentNullException() { ... }

[Fact]
public void Parse_ExceedsMaxLength_ThrowsArgumentException() { ... }

// combine with Theory for data-driven edge cases
[Theory]
[InlineData(null)]
[InlineData("")]
[InlineData("   ")]
public void Parse_BlankOrNullInput_ThrowsArgumentException(string input)
{
    Assert.Throws<ArgumentException>(() => Parser.Parse(input));
}
```

---

## 8. Independent

Tests must not depend on each other or assume any particular execution order. No shared mutable state between tests. If a test can only pass because a previous test ran first, it is hiding a real bug and will fail unpredictably in CI or when run in parallel.

```csharp
// bad — test 2 silently depends on test 1 having run first
private static Cart _sharedCart = new Cart();

[Fact]
public void Test1_AddItem_CartHasOneItem()
{
    _sharedCart.Add(new Item(price: 10m));
    Assert.Equal(1, _sharedCart.ItemCount);
}

[Fact]
public void Test2_GetTotal_ReturnsTen() // fails if run alone or before Test1
{
    Assert.Equal(10m, _sharedCart.GetTotal());
}

// good — each test owns its full setup
[Fact]
public void GetTotal_OneItemAdded_ReturnsTotalPrice()
{
    var cart = new Cart();
    cart.Add(new Item(price: 10m));

    Assert.Equal(10m, cart.GetTotal());
}
```

---

## 9. Readable (Arrange–Act–Assert)

A reader should understand the test without jumping to setup files or shared helpers. Arrange–Act–Assert is the universal pattern: set up the context, perform the action, verify the outcome. The three sections should always be clearly separated.

```csharp
[Fact]
public void GetTotal_OneItemAdded_ReturnsTotalPrice()
{
    // Arrange
    var cart = new Cart();
    cart.Add(new Item(price: 10m));

    // Act
    var total = cart.GetTotal();

    // Assert
    Assert.Equal(10m, total);
}
```

---

## 10. No Test Logic

Tests should not contain `if`, `for`, `switch`, or any other branching or looping constructs. Logic in a test means the test itself needs to be tested — and it obscures what is actually being verified. If you need to cover multiple scenarios, write multiple tests or use `[Theory]`.

```csharp
// bad — logic inside the test
[Fact]
public void Discount_AppliedCorrectly()
{
    var discounts = new[] { 0.1m, 0.2m, 0.5m };
    foreach (var rate in discounts)
    {
        var result = PriceCalculator.ApplyDiscount(100m, rate);
        Assert.Equal(100m * (1 - rate), result); // which iteration failed?
    }
}

// good — each case is explicit and independently reportable
[Theory]
[InlineData(0.1, 90)]
[InlineData(0.2, 80)]
[InlineData(0.5, 50)]
public void ApplyDiscount_ValidRate_ReturnsDiscountedPrice(
    decimal rate, decimal expected)
{
    var result = PriceCalculator.ApplyDiscount(100m, rate);
    Assert.Equal(expected, result);
}
```

---

## 11. Correct

A test that always passes gives false confidence and is worse than no test at all. The assertion must genuinely verify the behaviour described in the test name, and the test must be capable of failing when the production code is broken.

Common traps:

- **Asserting the wrong value** — passes by coincidence, not because the behaviour is correct
- **Swallowed exceptions** — a `try/catch` hides the failure the test was meant to catch
- **Testing the mock, not the code** — setting up a mock to return a value and then asserting that exact value, never exercising real logic
- **Missing `await`** — the assertion runs before the async operation completes, so it always passes vacuously

A reliable way to verify a test is correct is **mutation testing** — deliberately break the production code and confirm the test fails. If you can delete a line of production code and the test still passes, it is not really covering that behaviour.

```csharp
// bad — swallowed exception, test always passes
[Fact]
public void PlaceOrder_InsufficientStock_ThrowsInvalidOperationException()
{
    try
    {
        _orderService.PlaceOrder(new Order(quantity: 999));
    }
    catch
    {
        // exception swallowed — test never fails, even if no exception is thrown
    }
}

// bad — missing await, assertion runs before the operation completes
[Fact]
public async Task SaveUser_ValidUser_PersistsToDatabase()
{
    var user = new User("Ana");
    _userService.SaveAsync(user); // missing await — always passes vacuously

    var saved = await _repo.FindByNameAsync("Ana");
    Assert.NotNull(saved);
}

// bad — testing the mock, not the code
[Fact]
public void GetUser_ExistingId_ReturnsUser()
{
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.FindById(1)).Returns(new User { Name = "Ana" });

    var result = mockRepo.Object.FindById(1); // never calls UserService at all

    Assert.Equal("Ana", result.Name);
}

// good — straightforward assertion that will fail if the behaviour breaks
[Fact]
public void PlaceOrder_InsufficientStock_ThrowsInvalidOperationException()
{
    Assert.Throws<InvalidOperationException>(
        () => _orderService.PlaceOrder(new Order(quantity: 999)));
}
```

---

## Quick reference

| # | Attribute | Key question |
|---|-----------|-------------|
| 1 | Single Responsibility | Does this test verify exactly one logical concept? |
| 2 | Descriptive Name | Does it follow `TestedMethod_ScenarioTested_ExpectedBehaviour`? |
| 3 | Isolated | Does it avoid network, disk, and shared state? |
| 4 | Fast | Does it finish in milliseconds? |
| 5 | Tests Behaviour | Does it test the contract, not internals? |
| 6 | Deterministic | Does it give the same result every run? |
| 7 | Edge Cases | Does it cover null, empty, and boundary inputs? |
| 8 | Independent | Does it run correctly in any order, on its own? |
| 9 | Readable  | Is it clearly Arrange → Act → Assert? |
| 10 | No Test Logic | Is it free of `if`, `for`, and `switch` statements? |
| 11 | Correct | Will it actually fail when the production code is broken? |
