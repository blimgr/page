# Test Doubles Cheatsheet

> Five types, one purpose: replacing real dependencies in unit tests
>
> **Important:** these are *behaviours*, not rigid object types. A single test double object can exhibit multiple behaviours at once — e.g. stubbing some methods and spying on others.

## The Five Types

| Type | What it does | Wraps real impl? | Assert on it? | Typical use |
|------|-------------|------------------|---------------|-------------|
| **Dummy** | Passed around but never used. Just fills a required parameter slot. | No | No | `null` or empty object satisfying a constructor signature |
| **Fake** | A working but simplified implementation not suited for production. | No — it *is* its own impl | No | In-memory repository instead of a real database |
| **Stub** | Returns pre-programmed answers to control the state fed into the unit under test. | No | No | `getUser()` always returns a hardcoded user object |
| **Mock** | Expectations are set *upfront*. The framework fails the test if they are not met. | No by default — but can be configured | Yes — upfront | Declare `send()` must be called once before running the test |
| **Spy** | Records all calls silently. You inspect and assert *after* the fact. | Classically yes, but commonly wraps a stub too | Yes — after | Check `Received(1).Send()` after the unit runs |

## Key Distinction: Mock vs Spy

| | Mock | Spy |
|--|------|-----|
| **Style** | Prescriptive | Descriptive |
| **When expectations are set** | Before the test runs | After the test runs |
| **Who enforces?** | The framework automatically | You assert manually |
| **Mindset** | "This *must* happen" | "Here's what *happened*" |

## Stub and Spy/Mock are Orthogonal

These two concerns are independent and can be combined freely on the same object:

| Concern | Behaviour | Question it answers |
|---------|-----------|---------------------|
| **Stub** | Controls return values | What does the unit *receive*? |
| **Spy / Mock** | Verifies calls made | What did the unit *do*? |

A spy wrapping a stub is the most common pattern in everyday unit testing — especially when working against interfaces where there is no real implementation to wrap.

## .NET and JS Framework Styles

| Framework | Style | How |
|-----------|-------|-----|
| **NSubstitute** | Always spy | `Received()` after the fact. Wraps no real impl by default; use `ForPartsOf<T>()` to wrap a real class |
| **Moq — Loose** | Spy-like (default) | `Verify()` after the fact. Set `CallBase = true` to wrap real impl |
| **Moq — Strict** | Closer to mock | Fails on any unexpected call |
| **Jest (JS)** | Spy that reads like mock | `jest.fn()` records calls; `spyOn()` wraps real impl |
