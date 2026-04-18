# This reference in Typescript

## Introduction

A concept that is often misunderstood — especially by people with a background in languages that default to compile-time binding like Java or C# — is the concept of `this`. The confusion arises because in Java or C#, `this` always refers to the current instance of the class. In TypeScript, however, `this` can refer to different things depending on the context in which it is used, and that can lead to unexpected behavior and bugs. In this post, we will examine `this` in TypeScript and how we can mitigate the confusion around it.

## How `this` works

In a language that defaults to compile-time binding such as Java, C#, or C\+\+, the value of `this` is determined at compile time and always refers to the current instance of the class.

In contrast, in a language that defaults to runtime binding such as JavaScript and TypeScript, the value of `this` is determined at runtime based on how a function is called.

Let's look at the documentation about `this`. <a href="https://www.typescriptlang.org/docs/handbook/2/classes.html#this-at-runtime-in-classes" target="_blank">Typescript Documentation</a> says:

> It's important to remember that TypeScript doesn't change the runtime behavior of JavaScript, and that JavaScript is somewhat famous for having some peculiar runtime behaviors.

So it reminds us that TypeScript is a superset of JavaScript and does not change the runtime behavior of JavaScript. This means that the way `this` works in TypeScript is the same as in JavaScript, and that we need to understand how `this` works in JavaScript to understand how it works in TypeScript.

Let's look at what the <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this" target="_blank">JavaScript documentation</a> says about `this`:

> The value of `this` depends on in which context it appears: function, class, or global.

Already we can see that the meaning of `this` is more complex than just referring to the current instance of a class.

### JavaScript `this` Behavior Table (Corrected)

|#|Context|Scenario|What `this` refers to|
|:---|:---|:---|:---|
|1|**Function**|Plain function called as a method (`obj.fn()`)|The **receiver** (the object before the dot)|
|2|**Function**|Plain function called without an object|`undefined` (strict mode) / `globalThis` (non-strict mode)|
|3|**Function**|Function called with `call` / `apply`|The object passed as the first argument|
|4|**Function**|Function created with `bind`|The object passed to `bind` (permanently bound)|
|5|**Function**|Arrow function|Lexically captured from surrounding scope (at definition time)|
|6|**Class**|Instance method|The **receiver** (usually the instance) or `undefined` (strict mode) / `globalThis` (non-strict mode) if no receiver.|
|7|**Class**|Static method|The **receiver** (usually the class used to call it) or `undefined` (strict mode) / `globalThis` (non-strict mode) if no receiver.|
|8|**Class**|Arrow function property|Lexically captured `this` (in class fields: the instance)|
|9|**Class**|Constructor|The newly created instance|
|10|**Class**|Instance getter / setter|The **receiver** (usually the instance) or `undefined` (strict mode) / `globalThis` (non-strict mode) if no receiver.|
|11|**Class**|Static getter / setter|The **receiver** (usually the class used to call it) or `undefined` (strict mode) / `globalThis` (non-strict mode) if no receiver.|
|12|**Class**|Instance property initializer|The instance being created|
|13|**Class**|Static property initializer|The class (constructor)|
|14|**Class**|Static initialization block|The class (constructor)|
|15|**Global**|Global execution context|`globalThis`|

### Row-by-row breakdown

#### 1. Plain function called as a method

When a function is invoked with an object to the left of the dot, `this` is set to that object — the *receiver*. The function does not need to be defined on the object; what matters is how it is called.

```typescript
function greet(this: { name: string }) {
  console.log(`Hello, I am ${this.name}`);
}

const person = { name: "Alice", greet };
person.greet(); // "Hello, I am Alice" — `this` is `person`
```

#### 2. Plain function called without an object

When a function is called with no receiver, `this` is `undefined` in strict mode (TypeScript's default) or `globalThis` in non-strict mode. This is the source of most unexpected `undefined` bugs.

```typescript
function greet() {
  console.log(this); // TypeScript error under noImplicitThis; undefined at runtime
}

greet(); // no receiver
```

#### 3. Function called with `call` or `apply`

`call` and `apply` let you invoke a function immediately with an explicit `this`. The only difference is how arguments are passed: `call` takes them individually, `apply` takes them as an array.

```typescript
function greet(this: { name: string }, greeting: string) {
  console.log(`${greeting}, I am ${this.name}`);
}

const person = { name: "Alice" };

greet.call(person, "Hello");    // "Hello, I am Alice"
greet.apply(person, ["Hello"]); // "Hello, I am Alice"
```

#### 4. Function created with `bind`

`bind` returns a new function with `this` permanently fixed to the provided value, regardless of how or where the returned function is later called. Calling `bind` on an arrow function has no effect — arrow functions ignore `bind`.

```typescript
function greet(this: { name: string }) {
  console.log(`Hello, I am ${this.name}`);
}

const person = { name: "Alice" };
const boundGreet = greet.bind(person);

boundGreet();              // "Hello, I am Alice"
boundGreet.call({ name: "Bob" }); // Still "Hello, I am Alice" — bind wins
```

#### 5. Arrow function

Arrow functions do not have their own `this`. They capture `this` lexically from the enclosing scope at the time they are *defined*, not when they are called. This makes them predictable and immune to the receiver problem.

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

#### 6. Class instance method

Inside an instance method, `this` is the object the method was called on — the receiver. If the method is detached from the object and called as a plain function, the binding is lost and `this` becomes `undefined` in strict mode.

```typescript
class Counter {
  count = 0;
  increment() { this.count++; }
}

const counter = new Counter();
counter.increment(); // works — `this` is `counter`

const fn = counter.increment;
fn(); // Runtime error: cannot read properties of undefined
```

#### 7. Class static method

In a static method `this` refers to the receiver of the method which can be the class (constructor function) itself, a subclass or even undefined.

```typescript
class Base {
  static describe() { console.log(`I am ${this.name}`); }
}
class Child extends Base {}

Base.describe();  // "I am Base"
Child.describe(); // "I am Child"

const describe = Base.describe;
describe(); // Runtime error: cannot read properties of undefined (strict mode)
```

#### 8. Arrow function class field

A class field defined as an arrow function captures `this` lexically during construction — it always refers to the instance. This makes the method safe to detach and pass as a callback.

```typescript
class Timer {
  message = "Tick!";
  tick = () => console.log(this.message); // `this` is permanently the Timer instance
}

const timer = new Timer();
setTimeout(timer.tick, 1000); // Safe — `this` cannot be lost
```

#### 9. Constructor

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

#### 10. Instance getter / setter

Getters and setters on an instance behave like instance methods — `this` is the receiver. If the property descriptor is accessed through an unrelated object via `Object.getOwnPropertyDescriptor` and the getter is called directly, the receiver changes accordingly.

```typescript
class Box {
  private _value = 0;

  get value() {
    return this._value; // `this` is the Box instance
  }

  set value(v: number) {
    this._value = v; // `this` is the Box instance
  }
}

const box = new Box();
box.value = 42;
console.log(box.value); // 42
```

#### 11. Static getter / setter

Static getters and setters follow the same rule as static methods — `this` is the receiver of the property which can be the class (constructor) itself, a subclass or even undefined. Though to get the getter function itself, you need to access the property descriptor and call it directly.

```typescript
class Config {
  static _env = "production";

  static get env() {
    return this._env; // `this` is Config (or a subclass)
  }
}

class DevConfig extends Config {
  static _env = "development";
}

console.log(Config.env);    // "production"
console.log(DevConfig.env); // "development"
```

#### 12. Instance property initializer

When a class field (instance property) is initialised, `this` refers to the instance being constructed. Initialisers run as part of the constructor, in the order they are declared.

```typescript
class Greeter {
  name = "Alice";
  message = `Hello, I am ${this.name}`; // `this` is the Greeter instance
}

const g = new Greeter();
console.log(g.message); // "Hello, I am Alice"
```

#### 13. Static property initializer

When a static class field is initialised, `this` refers to the class (constructor function) itself, not an instance.

```typescript
class Registry {
  static label = "Registry";
  static description = `This is the ${this.label} class`; // `this` is Registry
}

console.log(Registry.description); // "This is the Registry class"
```

#### 14. Static initialization block

A static initialization block (`static { ... }`) runs once when the class is evaluated. Inside it, `this` refers to the class itself, making it useful for one-time setup that requires access to other static members.

```typescript
class AppConfig {
  static port: number;
  static host: string;

  static {
    this.port = parseInt(process.env.PORT ?? "3000", 10);
    this.host = process.env.HOST ?? "localhost";
    console.log(`Configured: ${this.host}:${this.port}`);
  }
}
```

#### 15. Global execution context

At the top level of a script (outside any function or class), `this` is `globalThis` — `window` in browsers, `global` in Node.js. In ES modules, the top-level `this` is `undefined`.

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

### Assigning a method to a function type

This can happen when you do an explicit assignment of a method to a variable with a function type or when you pass a method as an argument of function type. In both cases, the method loses its `this` context, because when it is called, there is no object to provide the context.

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

A more special and rare case of assignment is when you assign to a variable through destructuring:

```typescript
const { increment } = counter;
increment(); // Runtime error
```

Here we will also lose the `this` context and get a runtime error when we call `increment()`, because it is no longer being called as a method on the `counter` object.

**Fix: use an arrow function field**

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

**Alternative: bind in the constructor**

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

---

## Summary

The root of all `this` confusion in TypeScript is that `this` is a runtime concept, not a compile-time one. Its value is set by the **call site** — how and where the function is called — not where the function is defined. This is the fundamental difference from Java or C#.

### Common pitfalls and mitigations


|Pitfall|Root cause|TypeScript detection|Fix|
|:---|:---|:---|:---|
|Method passed as callback (`setTimeout`, event handler, etc.)|Implicit binding lost when detached from object|Not caught by default; add `this` parameter to detect|Arrow function field or `bind` in constructor|
|Destructuring a method `const { fn } = obj`|Same as above — method detached from object|Not caught by default|Arrow function field or `bind` in constructor|
|Nested regular function inside a method|Regular function creates its own `this` binding|`noImplicitThis` (part of `strict`) flags `this` as implicit `any`|Replace nested function with an arrow function|
|Using `this` before `super()` in a subclass constructor|Parent class has not yet initialised the instance|Hard compile-time error in TypeScript|Call `super()` first|
|Expecting static methods to be fixed to the defining class|Static `this` is dynamic — refers to the class it is called on|No error; intentional behaviour|Use intentionally for factory patterns; document the expectation|

TypeScript does not solve the problem at the language level, but it gives you tools — the `this` parameter and strict mode — to catch misuse early. Understanding where `this` is set, and applying the patterns shown above, is all you need to write TypeScript that behaves exactly as you expect.