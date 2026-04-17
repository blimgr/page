# Constants Don’t Fix Magic Values — Names Do

## Why This Post

One of the first rules we learn in programming is: **avoid magic values**. Hardcoding arbitrary numbers or strings directly into logic is considered a code smell.

But this rule is often applied mechanically. You end up with code like this:

```csharp
const int Value_1 = 1;
const int Value_Zero = 0;
const string Value_Active = "active";
```

Technically, the literals are gone—but the problem remains.

This post is about the real goal of the rule: **making code understandable**, not just “hiding” literals.

---

## What Is a Magic Value?

A magic value is a literal (like `86400`, `"A"`, or `0.05`) that appears in code without explanation. It’s “magic” because its meaning isn’t obvious to the reader.

```csharp
// What does 86400 mean?
if (elapsed > 86400)
{
    ExpireSession();
}

// What does "A" represent?
if (user.Status == "A")
{
    SendWelcomeEmail(user);
}
```

These values carry hidden meaning. Without context, a reader has to guess—or dig.

---

## The Naive Rule

A common interpretation is:

> *Replace every literal with a named constant.*

This leads to code that is technically compliant—but not more readable.

---

## The Real Rule

The goal is not to replace literals.  
The goal is to **reveal meaning**.

When you extract a constant, you’re making a promise:

> *This name explains something the raw value cannot.*

A name like `Value_86400` breaks that promise. It tells you nothing new.

Compare:

```csharp
const int Value_86400 = 86400;
const string Value_A = "A";
```

vs.

```csharp
const int SecondsInADay = 86400;
const string ActiveUserStatus = "A";
```

Now the code communicates intent:

```csharp
if (elapsed > SecondsInADay)
{
    ExpireSession();
}

if (user.Status == ActiveUserStatus)
{
    SendWelcomeEmail(user);
}
```

You understand *why* the value exists—not just what it is.

---

## Constants vs Modeling

Sometimes, a well-named constant is still not the best solution.

If a value represents a fixed set of states, consider modeling it explicitly:

```csharp
enum UserStatus
{
    Active,
    Inactive
}
```

This removes the magic value entirely instead of just renaming it.

Constants improve readability.  
Good modeling eliminates ambiguity.

---

## “But Constants Prevent Typos”

That’s true—and useful.

But it’s not enough.

A constant should:
1. **Prevent duplication**
2. **Prevent typos**
3. **Explain meaning**

If it only does the first two, you’ve solved a mechanical problem but left the semantic one intact.

---

## Good and Bad Examples

### Bad: Names that restate the value

```csharp
const int Value_Zero = 0;
const int Value_One = 1;
const bool Value_True = true;
const string Value_Empty = "";
```

These add noise without adding meaning.

---

### Good: Names that explain intent

```csharp
const int MaxLoginAttempts = 3;
const int MinPasswordLength = 8;
const string DefaultCountryCode = "GR";
const string GuestUserRole = "guest";
```

Now the values tell a story.

---

### Bad: Replacing meaningful literals with meaningless names

```csharp
const string Str1 = "application/json";
const string Str2 = "Bearer";

request.Headers.Add(Str2 + " " + token);
```

You’ve reduced clarity.

---

### Good: Adding context

```csharp
const string JsonContentType = "application/json";
const string BearerAuthScheme = "Bearer";

request.Headers.Add($"{BearerAuthScheme} {token}");
```

Now the code reads like documentation.

---

## What About `0`, `1`, `true`, and `false`?

These are often over-abstracted.

In most cases, they are already clear:

- `0` → nothing / start
- `1` → single unit
- `true` / `false` → boolean state

This is unnecessary:

```csharp
const int Zero = 0;
const int One = 1;
const bool Yes = true;

for (int i = Zero; i < items.Count; i += One)
{
    items[i].IsVisible = Yes;
}
```

The original version is clearer.

---

### When They *Do* Need Names

When the value carries domain meaning:

```csharp
// What does index 0 represent?
var primaryAccount = user.Accounts[0];

// Clearer
const int PrimaryAccountIndex = 0;
var primaryAccount = user.Accounts[PrimaryAccountIndex];
```

```csharp
// Why exactly 1?
if (order.Items.Count == 1)
{
    ApplySingleItemDiscount(order);
}

// Intent is explicit
const int SingleItemThreshold = 1;
if (order.Items.Count == SingleItemThreshold)
{
    ApplySingleItemDiscount(order);
}
```

```csharp
// Why true?
user.IsEmailVerified = true;

// Now we know
const bool DefaultEmailVerificationState = true; // e.g. SSO users
user.IsEmailVerified = DefaultEmailVerificationState;
```

---

## A Simple Test

Ask yourself:

> **Does this name tell me something the value itself does not?**

- If **yes** → extract it
- If **no** → either rename it better or leave the literal

---

## Summary

The magic value rule is about **semantics**, not syntax.

Replacing a literal with a meaningless name doesn’t remove the magic—it just hides it.

> **If the name doesn’t add meaning, you didn’t fix the problem—you moved it.**
