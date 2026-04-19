# The Many Faces of `this` in Typescript

## Introduction

A concept that is often misunderstood — especially by people with a background in languages like Java, C# or C++ — is the concept of `this` in TypeScript/JavaScript. 

The confusion arises because in Java, C# or C++, `this` is lexically bound to the current instance and cannot change based on how a method is called.  

In TypeScript, however, `this` can refer to different things depending on the context in which it is used, and that can lead to unexpected behavior and bugs. In this post, we will examine `this` in TypeScript and how we can mitigate the confusion around it.

## How `this` works

In languages like Java or C#, this is lexically bound to the current instance and cannot change based on how a method is called.

In contrast, in JavaScript and TypeScript, the value of `this` is determined at runtime based on how a function is called.

Let's look at the documentation about `this`. <a href="https://www.typescriptlang.org/docs/handbook/2/classes.html#this-at-runtime-in-classes" target="_blank">Typescript Documentation</a> says:

> It's important to remember that TypeScript doesn't change the runtime behavior of JavaScript, and that JavaScript is somewhat famous for having some peculiar runtime behaviors.

So it reminds us that TypeScript is a superset of JavaScript and does not change the runtime behavior of JavaScript. This means that the way `this` works in TypeScript is the same as in JavaScript, and that we need to understand how `this` works in JavaScript to understand how it works in TypeScript.

Let's look at what the <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this" target="_blank">JavaScript documentation</a> says about `this`:

> The value of `this` depends on in which context it appears: function, class, or global.

Already we can see that the meaning of `this` is more complex than just referring to the current instance of a class.

For functions (either they are plain functions, methods, setters, getters, static or instance), the value of `this` is determined by how the function is called. If it is called as a method of an object, `this` refers to that object (receiver of the method). 

If it is called as a plain function, `this` is `undefined` in strict mode or `globalThis` in non-strict mode. 

If it is called with `call`, `apply`, or `bind`, `this` is explicitly set to the first argument passed to those methods. 

Arrow functions capture `this` lexically from the surrounding scope at the time they are defined.

In class constructors and in instance property initializers, `this` refers to the newly created instance. 

In static initialization blocks and static property initializers, `this` refers to the class itself.

In global scope, `this` always refers to `globalThis` (With exception to ES modules, where `this` is `undefined`).

### `this` Behavior
Case|What `this` refers to| Section |
|:---|:---|---:|
| Function called as a method (`obj.fn()`)|The receiver (the object before the dot)|1.a. Functions - Receiver|
| Function called without an object|`undefined` (strict mode) / `globalThis` (non-strict mode)|1.b. Functions - No Receiver|
| Function called with `call` / `apply`|The object passed as the first argument|1.c. Functions - Usage of `call`, `apply` or `bind`|
| Function created with `bind`|The object passed to `bind` (permanently bound)|1.c. Functions - Usage of `call`, `apply` or `bind`|
| Arrow function|Lexically captured from surrounding scope (at definition time)|2. Arrow Functions|
| Class constructor|The newly created instance| 3. Constructors and Static Initialization Blocks|
| Static initialization block|The class (constructor)|3. Constructors and Static Initialization Blocks|
| Instance property initializer|The instance being created|4. Property Initializers|
| Static property initializer|The class (constructor)|4. Property Initializers|
| Global scope|`globalThis`|5. Global Scope|



### 1. Functions
#### a. Receiver
When a function (plain function, method, getter¹, setter¹, static² or instance) is called on an instance of a class, `this` is the instance (the receiver). 

Note that the receiver is determined at the call site, not where the function is defined. This means that you can define a method on one class and call it on an instance of another class, and `this` will refer to the instance that is calling the method, not the class where the method was defined.

¹ Getters and setters are special methods that are defined using the `get` and `set` keywords. They are called as properties, but they are still functions under the hood, and they have a receiver when called. To get the function object of a getter or setter, you can use `Object.getOwnPropertyDescriptor` to access the descriptor on the object prototype for instance properties and the class constructor for static properties, and then get the `get` or `set` property from it.  
² Static methods are called on the class itself, not on an instance. However, they still have a receiver — the class (constructor function) they are called on.

```typescript
class Person {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  greet() {
    console.log(`Hello, I am ${this.name}`);
  }
}

class Employee extends Person {
  jobTitle: string;
  constructor(name: string, jobTitle: string) {
    super(name);
    this.jobTitle = jobTitle;
  }
}

const alice = new Person("Alice");
alice.greet(); // "Hello, I am Alice" — `this` is `alice`
const bob = new Employee("Bob", "Developer");
bob.greet(); // "Hello, I am Bob" — `this` is `bob`

bob.greet = alice.greet; // Method from alice assigned to bob
bob.greet(); // "Hello, I am Bob" — `this` is still `bob` because of the receiver
```
#### b. No Receiver
When a function (plain function, method, getter, setter, static or instance) is assigned¹, if it has any receiver, it is lost.
If it is then called via that assignment without an object `this` becomes `undefined` in strict mode or `globalThis` in non-strict mode.

¹ This can happen through an explicit assignment to a variable with a function type, through passing the method as an argument of function type, or through destructuring assignment, etc.

```typescript
class Counter {
  count = 0;
  increment() { this.count++; }
}
const counter = new Counter();
const increment = counter.increment; // plain assignment
increment(); // TypeError: Cannot read property 'count' of undefined

const { increment: inc } = counter; // destructuring assignment
inc(); // TypeError: Cannot read property 'count' of undefined

setTimeout(counter.increment, 1000); // callback assignment — TypeError: Cannot read property 'count' of undefined
```
#### c. Usage of `call`, `apply` or `bind`
When a function (plain function, method, getter, setter, static or instance) is called with `call` or `apply`, the first argument explicitly sets the value of `this` for that call.

```typescript
class Counter {
  constructor(public count = 0) {}
  increment() { 
    this.count++; 
    console.log(this.count); 
    }
}

const counter = new Counter(5);

const increment = counter.increment;
increment.call(new Counter(10)); // 11 — `this` is new Counter(10)
increment.apply(new Counter(10)); // 11 — `this` is new Counter(10)

const { increment: inc } = counter;
inc.call(new Counter(10)); // 11 — `this` is new Counter(10)
inc.apply(new Counter(10)); // 11 — `this` is new Counter(10)
```

When a function (plain function, method, getter, setter, static or instance) is called with `bind`, it returns a new function with `this` permanently set to the first argument used in `bind`. Note that a bound function cannot be re-bound with `call` or `apply` — the `this` value will always be the one provided to `bind`.

```typescript
class Counter {
  constructor(public count = 0) {
      this.increment = this.increment.bind(this);
  }

  increment() { 
    this.count++; 
    console.log(this.count); 
  }
}
const counter = new Counter();
counter.increment(); // works — `this` is `counter`
const increment = counter.increment;
increment(); // works — `this` is still `counter` because of bind
const { increment: inc } = counter;
inc(); // works — `this` is still `counter` because of bind
setTimeout(counter.increment, 1000); // works — `this` is still `counter` because of bind
```
### 2. Arrow Functions

Arrow functions do not have their own `this`. They capture `this` lexically from the enclosing scope at the time they are *defined*, not when they are called. This makes them predictable and avoids dynamic rebinding issues.

```typescript
const person = {
  name: "Alice",
  greet() {
    const inner = () => console.log(`Hello, I am ${this.name}`);
    inner(); // `this` is whatever `greet`'s `this` is
  },
};

person.greet(); // "Hello, I am Alice"
```
This makes the arrow functions behave like methods that are permanently bound using `bind`. For instance properties set to arrow functions there is a difference: they are not on the prototype, but each instance gets its own copy of the function.

```typescript
class Counter {

  constructor() {
    this.decrement = this.decrement.bind(this);
  }
  count = 0;
  increment = () => {
    this.count++;
    console.log(this.count);
  };

  decrement() {
    this.count--;
    console.log(this.count);
  }
}

console.log(Counter.prototype.decrement); // [Function: decrement]
console.log(Counter.prototype.increment); // undefined
```
### 3. Constructors and Static Initialization Blocks

Inside a constructor, `this` is the newly created instance. In a subclass, `this` is not accessible until `super()` has been called — attempting to use `this` before `super()` is a hard error in both TypeScript and at runtime.

```typescript
class Animal {
  name: string;
  constructor(name: string) {
    this.name = name; // `this` is the new Animal instance
  }
}

class Dog extends Animal {
  breed: string;
  constructor(name: string, breed: string) {
    // console.log(this.name); // Error: must call super first
    super(name);
    this.breed = breed; // safe now
  }
}
```
Static initialization blocks (`static { ... }`) also have `this` bound to the class (constructor function) they are defined in, and they run once when the class is evaluated.

```typescript
class AppConfig {
  static port: number;
  static host: string;

  static {
    this.port = parseInt(process.env.PORT ?? "3000", 10); // `this` is the AppConfig class
    this.host = process.env.HOST ?? "localhost"; // `this` is the AppConfig class
    console.log(`Configured: ${this.host}:${this.port}`); // `this` is the AppConfig class
  }
}
```
### 4. Property Initializers

When a class field (instance property) is initialized, `this` refers to the instance being constructed. Initializers run as part of the constructor, in the order they are declared.

```typescript
class Greeter {
  name = "Alice";
  message = `Hello, I am ${this.name}`; // `this` is the Greeter instance
}

const g = new Greeter();
console.log(g.message); // "Hello, I am Alice"
```

When a static property is initialized, `this` refers to the class (constructor function) it is defined in.

```typescript
class AppConfig {
  static port = parseInt(process.env.PORT ?? "3000", 10);
  static host = process.env.HOST ?? "localhost";
  static description = `Configured: ${this.host}:${this.port}`; // `this` is the AppConfig class
}
```
### 5. Global Scope

At the top level of a script (outside any function or class), `this` is `globalThis`.

```typescript
// In a CommonJS/script context:
console.log(this === globalThis); // true

// In an ES module:
console.log(this); // undefined
```


## TypeScript-specific additions

TypeScript extends JavaScript's `this` story with a few features that give you better type safety.

### The `this` parameter

TypeScript allows you to declare a fake first parameter named `this` in a function signature. This parameter is erased at compile time and does not affect the JavaScript output; it exists purely to tell TypeScript what type `this` must be when the function is called.

```typescript
interface User {
  name: string;
  greet(this: User): void;
}

const user: User = {
  name: "Alice",
  greet(this: User) {
    console.log(`Hello, I am ${this.name}`);
  },
};

user.greet(); // OK

const fn = user.greet;
fn(); // TypeScript error: The 'this' context of type 'void' is not assignable to method's 'this' of type 'User'.
```

This is a powerful tool: it catches the classic `this`-losing bug at compile time, rather than letting it slip through to runtime.

## Common pitfalls

### Assigning a method

When a method is assigned, it loses its `this` context. When subsequently called via the assignment without a receiver, `this` becomes `undefined` in strict mode or `globalThis` in non-strict mode, leading to runtime errors when the method tries to access properties on `this`.

This is the single most common `this` bug in TypeScript.

You define a class, write a method, and then pass it as a callback — and `this` vanishes.

```typescript
class Timer {
  message = "Tick!";

  start() {
    setTimeout(this.tick, 1000); // Bug: `this` inside tick will be undefined
  }

  tick() {
    console.log(this.message);
  }
}
```

When `setTimeout` eventually calls `tick`, it calls it as a plain function. The implicit binding to the `Timer` instance is lost.

The callback can be an event handler, a promise callback, an array method callback, or any other function that expects a function argument.

A more subtle and less obvious case of assignment is when you assign to a variable through destructuring:

```typescript
const { increment } = counter;
increment(); // Runtime error
```

Here we will also lose the `this` context and get a runtime error when we call `increment()`, because it is no longer being called as a method on the `counter` object.

**Protection : use this parameter to catch at compile time**

In order to protect against such bugs, you can add a `this` parameter to the method signature. This will cause TypeScript to type check that the method is always called with the correct `this` type, and it will give you a compile-time error if you try to call it with the wrong type. 

This parameter not only protects against the common case of passing the method as a callback, but also against any other case where the method is called without the correct receiver.

```typescript
class Timer {
  message = "Tick!";

  start() {
    setTimeout(this.tick, 1000); // TypeScript error: The 'this' context of type 'void' is not assignable to method's 'this' of type 'Timer'.
  }

  tick(this: Timer) {
    console.log(this.message);
  }
}
```
**Fix: use an arrow function field**

However, adding a `this` parameter only catches the bug at compile time. It does not fix the underlying issue that the method loses its `this` context when passed as a callback and it is uncommon and a bit verbose to add a `this` parameter to every method just to catch this bug at compile time. You still need to fix the code to ensure that `this` is correctly bound when the method is called.

The cleanest solution is to define the method as a class field using arrow function syntax. `this` is then lexically bound to the instance at construction time and can never be lost.

```typescript
class Timer {
  message = "Tick!";

  tick = () => {
    console.log(this.message); // Always the Timer instance
  };

  start() {
    setTimeout(this.tick, 1000); // Safe to pass as callback
  }
}
```

The trade-off is that arrow function fields are not on the prototype — each instance gets its own copy of the function, which uses slightly more memory. For most applications this is irrelevant.

You can also use an arrow function directly in the callback.

```typescript
class Timer {
  message = "Tick!";

  tick() {
    console.log(this.message);
  }

  start() {
    setTimeout(() => this.tick(), 1000); // Arrow function captures `this` from start()
  }
}
```

**Alternative Fix: bind in the constructor**

If you prefer to keep the method on the prototype, explicitly bind it in the constructor:

```typescript
class Timer {
  message = "Tick!";

  constructor() {
    this.tick = this.tick.bind(this);
  }

  tick() {
    console.log(this.message);
  }

  start() {
    setTimeout(this.tick, 1000); // Safe
  }
}
```

### Nested regular functions inside methods

A regular function defined inside a method creates its own `this` binding, severing any connection to the outer method's `this`. This catches people off guard because the nested function *looks* like it belongs to the method. In TypeScript with `strict` mode enabled, `noImplicitThis` (which is part of the `strict` bundle) catches this at compile time, because TypeScript cannot determine what `this` refers to inside the nested function and would otherwise type it as `any`.

```typescript
class Processor {
  label = "Processor";

  run(items: string[]) {
    items.forEach(function (item) {
      console.log(`${this.label}: ${item}`); // TypeScript error: 'this' implicitly has type 'any'
    });
  }
}
```

**Fix: replace with an arrow function**

Replacing the nested function with an arrow function preserves the outer method's `this`:

```typescript
class Processor {
  label = "Processor";

  run(items: string[]) {
    items.forEach((item) => {
      console.log(`${this.label}: ${item}`); // `this` is correctly the Processor instance
    });
  }
}
```

### `this` in static methods and subclasses

In a static method, `this` refers to the class itself, not an instance. That alone surprises many developers. What surprises them even more is what happens with inheritance: when a subclass inherits a static method, `this` inside that method becomes the subclass, not the parent class where the method was originally defined.

```typescript
class Base {
  static create() {
    return new this(); // `this` is the class, not an instance
  }

  static describe() {
    console.log(`I am ${this.name}`);
  }
}

class Child extends Base {}

Base.describe();  // "I am Base"
Child.describe(); // "I am Child" — `this` is Child, not Base
Child.create();   // returns a Child instance, not a Base instance
```

This dynamic `this` in static methods is actually useful for factory patterns — `create()` above will always return an instance of whatever class you call it on. But it is deeply counterintuitive if you expect static methods to be fixed to the class where they were written.

### Pitfall Quick Reference
|Pitfall|Root cause|TypeScript detection|Fix|
|:---|:---|:---|:---|
|Method passed as callback (`setTimeout`, event handler, etc.)|Implicit binding lost when detached from object|Not caught by default; add `this` parameter to detect|Arrow function field or `bind` in constructor|
| Assignment to a variable with function type|Same as above — method detached from object|Not caught by default; add `this` parameter to detect|Arrow function field or `bind` in constructor|
|Destructuring a method `const { fn } = obj`|Same as above — method detached from object|Not caught by default; add `this` parameter to detect|Arrow function field or `bind` in constructor|
|Nested regular function inside a method|Regular function creates its own `this` binding|`noImplicitThis` (part of `strict`) flags `this` as implicit `any`|Replace nested function with an arrow function|
|Using `this` before `super()` in a subclass constructor|Parent class has not yet initialized the instance|Hard compile-time error in TypeScript|Call `super()` first|
|Expecting static methods to be fixed to the defining class|Static `this` is dynamic — refers to the class it is called on|No error; intentional behaviour|Use intentionally for factory patterns; document the expectation|

## Summary

The root of all `this` confusion in TypeScript is that `this` is a runtime concept, not a compile-time one. Its value is set by the **call site** — how and where the function is called — not where the function is defined. This is a fundamental difference from Java or C# that when you grasp it, you can understand and predict how `this` behaves in any context in TypeScript and prevent undesirable bugs and unpredictable behavior.