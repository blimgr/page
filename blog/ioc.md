# Inversion of Control (IoC): Dependency Injection Explained

## Introduction
In previous <a href="#/blog/dip">post</a>, we have explored the Dependency Inversion Principle (DIP) and how it differs from Dependency Injection (DI). However, any discussion of these topics would be incomplete without also addressing Inversion of Control (IoC) and DI containers.

All of these terms are important in software engineering, widely used, and often misunderstood or used interchangeably. This post aims to clarify their meaning and disambiguate them.

It begins by establishing clear definitions of IoC and DI containers, building on the definitions already introduced for DIP and DI. It then explores how these concepts relate to one another.

While the relationship between DIP and DI has already been examined in previous <a href="#/blog/dip">post</a>, this post focuses on how DI relates to DI containers and where IoC fits into the overall picture.

## Inversion of Control (IoC)
Inversion of Control (IoC) is a high-level design principle in which, contrary to traditional control flow where application code directs execution and manages dependencies, control over aspects of a program’s behavior—such as object creation, execution flow, and lifecycle management—is delegated to an external orchestrator. This principle is independent of specific implementations and can be realized through various techniques (e.g., dependency injection, event-driven callbacks, or framework-driven execution).

Something you may have come across is the well known Hollywood saying:
“Don’t call us, we’ll call you.”

### Dimensions of Control Inversion

| Aspect of Control     | Traditional Approach                                      | With IoC                                              | Controlled By            | Common Mechanisms                         |
|----------------------|----------------------------------------------------------|------------------------------------------------------|--------------------------|-------------------------------------------|
| Object creation      | Application code instantiates dependencies (`new`)       | Dependencies are provided externally                 | DI container / factory   | Dependency Injection                      |
| Execution flow       | Application code decides what runs and when              | Framework/runtime invokes application code           | Framework / runtime      | Framework lifecycle, controllers          |
| Lifecycle management | Application code manages object lifetime                 | Lifecycle handled externally                         | Container / runtime      | Scoped, singleton, transient lifetimes    |
| Event handling       | Application code directly calls logic or polls           | Code reacts to externally triggered events           | Event system / runtime   | Callbacks, observers, event emitters      |
| Algorithm structure  | Full control over algorithm steps                        | High-level flow fixed, steps delegated               | Base class / framework   | Template Method pattern                   |
| Module wiring        | Dependencies manually composed in code                   | Composition defined externally                       | Container / configuration| Composition root, DI configuration        |


### Dependency Inversion and Inversion of Control
What is the connection between DIP and IoC? This was a question on the previous 
<a href="#/blog/dip">post</a>. As soon as you apply DIP, something has to be responsible for providing the concrete implementations behind your abstractions — and that something is always a form of IoC. That could be:
* A DI container resolving constructor dependencies
* A factory creating the concrete instance
* A service locator looking it up at runtime
* A template method

In all four cases you've respected DIP — your high-level module knows nothing about the concrete type — and in all three cases some form of IoC is what makes that possible at runtime. DIP without IoC leaves you with well-structured abstractions you can't actually resolve.

## Dependency Injection and DI Container (DI)

Dependency Injection (DI) is a technique for implementing Inversion of Control by inverting how a class obtains its dependencies. Instead of a class creating its own dependencies with new, they are provided externally—most commonly through constructor injection. This decouples the class from the concrete implementations it depends on, making it easier to test, replace, and reason about in isolation.

Importantly, DI does not require any framework or tooling. You can practice Pure DI by manually composing your object graph at the entry point of your application—the Composition Root:

```csharp
// Composition Root — no tooling, just manual wiring
var db = new PostgresDatabase();
var repo = new UserRepository(db);
var service = new OrderService(repo);
var controller = new OrderController(service);
```

Classes still receive their dependencies from the outside, still depend on abstractions, and remain unaware of how they are wired. The application itself acts as the composer. This approach works well for small systems, but as the object graph grows, manual composition becomes harder to maintain—this is where tooling becomes valuable.

A DI container—sometimes also called an IoC container—is a runtime tool that automates dependency injection at scale. In addition to supplying dependencies, a DI container typically takes over responsibilities that would otherwise be handled manually:

* Object creation — instead of calling new, the container instantiates objects when needed
* Lifecycle management — instead of managing lifetimes manually, the container controls them (e.g. transient, scoped, singleton)
* Dependency wiring — instead of manually composing the object graph, the container resolves it automatically based on declared dependencies

You register your types and their lifetimes once—e.g. `builder.Services.AddScoped<IDatabase, PostgresDatabase>()`—and from that point on the container handles creation, wiring, and lifecycle whenever that service is requested.

## Summary 
| Concept | Formal Definition | Relationship to Others |
| :--- | :--- | :--- |
| **Inversion of Control (IoC)** | A broad design principle where the flow of control is delegated to an external framework or orchestrator. | **The Umbrella:** IoC is the foundational principle. Every other item in this table is a specific application or tool used to realize IoC. |
| **Dependency Inversion Principle (DIP)** | A high-level architectural strategy stating that both high and low-level modules must depend on abstractions (interfaces). | **The Goal:** DIP defines *how* dependencies should be structured. It **requires** IoC because you cannot invert a dependency if your code still controls instantiation. |
| **Dependency Injection (DI)** | A design pattern where a component's dependencies are provided by an external caller rather than created by the component itself. | **The Pattern:** A specific subset of IoC. It is the "delivery mechanism" for DIP. While DIP says "depend on an interface," DI says "here is the object that implements that interface." |
| **DI / IoC Container** | A library or framework that automates the instantiation, lifecycle management, and wiring of objects. | **The Tool:** The engine that performs DI at scale. It handles the "plumbing" so developers don't have to manually instantiate and pass objects. |