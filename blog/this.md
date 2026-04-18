# This reference in Typescript

## Introduction

A concept that is often misunderstood — especially by people with a background in languages that default to compile-time binding like Java or C# — is the concept of `this`. The confusion arises because in Java or C#, `this` always refers to the current instance of the class. In TypeScript, however, `this` can refer to different things depending on the context in which it is used, and that can lead to unexpected behavior and bugs. In this post, we will examine `this` in TypeScript and how we can mitigate the confusion around it.

## How `this` works

In a language that defaults to compile-time binding such as Java, C#, or C++, the value of `this` is determined at compile time and always refers to the current instance of the class.

In contrast, in a language that defaults to runtime binding such as JavaScript and TypeScript, the value of `this` is determined at runtime based on how a function is called.

Let's look at the documentation about `this`. <a href="https://www.typescriptlang.org/docs/handbook/2/classes.html#this-at-runtime-in-classes" target="_blank">Typescript Documentation</a> says:

> It's important to remember that TypeScript doesn't change the runtime behavior of JavaScript, and that JavaScript is somewhat famous for having some peculiar runtime behaviors.

So it reminds us that TypeScript is a superset of JavaScript and does not change the runtime behavior of JavaScript. This means that the way `this` works in TypeScript is the same as in JavaScript, and that we need to understand how `this` works in JavaScript to understand how it works in TypeScript.

Let's look at what the <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this" target="_blank">JavaScript documentation</a> says about `this`:

> The value of `this` depends on in which context it appears: function, class, or global.

Already we can see that the meaning of `this` is more complex than just referring to the current instance of a class.

### Function context

There are four main ways a function can be called, and each one sets `this` differently.

#### 1. Default binding (global or `undefined`)

When a function is called as a plain function — not as a method, not with `new` — `this` falls back to the global object in non-strict mode (`window` in browsers, `global` in Node.js), or `undefined` in strict mode.

```typescript
function greet() {
  console.log(this); // window (browser) or undefined (strict mode) and compilation error in TypeScript
}

greet();
```

TypeScript projects almost always run in strict mode, which means calling a function this way and relying on `this` will throw a runtime error. This is usually the first surprise for developers coming from Java or C#.

#### 2. Implicit binding (method call)

When a function is called as a method on an object, `this` refers to that object — the object to the left of the dot at the call site.

```typescript
const person = {
  name: "Alice",
  greet() {
    console.log(`Hello, I am ${this?.name}`);
  },
};

person.greet(); // "Hello, I am Alice"
```

The key word here is *call site*. It does not matter where `greet` was defined; what matters is how it was called. This is the heart of the confusion:

```typescript
const greetFn = person.greet;
greetFn(); // "Hello, I am undefined" — `this` is now the global object!
```

By pulling `greet` out of the object and calling it as a plain function, we lost the implicit binding. `this` is no longer `person`.

#### 3. Explicit binding (`call`, `apply`, `bind`)

JavaScript provides three built-in methods to explicitly set `this` when calling a function.

`call` and `apply` invoke the function immediately, with `this` set to the first argument. The difference is in how you pass the remaining arguments: `call` takes them as a comma-separated list, while `apply` takes them as an array.


```typescript
function greet(this: { name: string }, greeting: string) {
  console.log(`${greeting}, I am ${this.name}`);
}

const person = { name: "Alice" };

greet.call(person, "Hello");    // "Hello, I am Alice"
greet.apply(person, ["Hello"]); // "Hello, I am Alice"
```

`bind` does not invoke the function immediately. Instead, it returns a new function with `this` permanently bound to the provided value, no matter how or where it is called later.

```typescript
const boundGreet = greet.bind(person);
boundGreet("Hello"); // "Hello, I am Alice"
```

### Arrow functions

Arrow functions are special. They do **not** have their own `this` binding. Instead, they lexically capture `this` from the surrounding scope at the time they are defined — not at the time they are called.

```typescript
const person = {
  name: "Alice",
  greet() {
    const inner = () => {
      console.log(`Hello, I am ${this.name}`);
    };
    inner();
  },
};

person.greet(); // "Hello, I am Alice"
```

Here, `inner` is an arrow function. When it references `this`, it looks up to its enclosing scope — the `greet` method — and uses whatever `this` is there. Since `greet` was called as a method on `person`, `this` inside `greet` (and therefore inside `inner`) is `person`.

Contrast this with a regular function:

```typescript
const person = {
  name: "Alice",
  greet() {
    const inner = function () {
      console.log(`Hello, I am ${this.name}`); // undefined in strict mode and compilation error in TypeScript
    };
    inner();
  },
};

person.greet();
```

The regular `inner` function creates its own `this` binding. Since it is called as a plain function (no object to the left of the dot), its `this` is `undefined` in strict mode.

### Class context

#### 1. Methods
In TypeScript classes, `this` inside a method refers to the instance of the class — but only when the method is called as a method. If you extract the method and call it as a plain function, the same problem applies.

```typescript
class Counter {
  count = 0;

  increment() {
    this.count++;
  }
}

const counter = new Counter();
counter.increment(); // works fine

const fn = counter.increment;
fn(); // Runtime error: Cannot set properties of undefined
```
#### 2. Constructors
Inside a constructor, `this` refers to the new instance being created — straightforward enough. The gotcha appears in subclass constructors: you must call `super()` before accessing `this`. Attempting to use `this` before `super()` is a hard error both at runtime and in TypeScript.

```typescript
class Animal {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}

class Dog extends Animal {
  breed: string;
  constructor(name: string, breed: string) {
    console.log(this.name); // TypeScript error: 'super' must be called before accessing 'this'
    super(name);
    this.breed = breed;
  }
}
```

The reason is that until `super()` runs, the parent class has not had a chance to initialise the instance. `this` does not yet exist in a usable state, so the language forbids touching it.

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

### The `ThisType<T>` utility type

`ThisType<T>` is a built-in utility type that lets you set the type of `this` across all methods in an object literal at once — without annotating each method individually. It is commonly used when building object factories or mixin patterns.

```typescript
type Methods = {
  greet(): void;
};

type State = {
  name: string;
};

const methods: Methods & ThisType<State> = {
  greet() {
    console.log(`Hello, I am ${this.name}`); // `this` is typed as State
  },
};
```

TypeScript infers `this` as `State` inside every method in the object, without any per-method `this` parameter annotation. Like the `this` parameter, it is erased at runtime — it is purely a compile-time signal.


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

### `this` resolution by calling pattern

| Calling pattern | Value of `this` | Notes |
|---|---|---|
| Plain function call `fn()` | `undefined` (strict) / global object | Strict mode is default in TypeScript |
| Method call `obj.fn()` | `obj` | The object to the left of the dot |
| `fn.call(ctx, ...)` | `ctx` | Invoked immediately |
| `fn.apply(ctx, [...])` | `ctx` | Invoked immediately, args as array |
| `fn.bind(ctx)()` | `ctx` | Returns a new function, permanent |
| `new Fn()` | New instance | A fresh object is created |
| Arrow function | Lexical `this` from enclosing scope | Set at definition time, not call time |
| Static method `Class.fn()` | `Class` itself | In subclasses, `this` is the subclass |
| Arrow function + `bind` | Lexical `this` (unchanged) | `bind` has no effect on arrow functions |

### Common pitfalls and mitigations

| Pitfall | Root cause | TypeScript detection | Fix |
|---|---|---|---|
| Method passed as callback (`setTimeout`, event handler, etc.) | Implicit binding lost when detached from object | Not caught by default; add `this` parameter to detect | Arrow function field or `bind` in constructor |
| Destructuring a method `const { fn } = obj` | Same as above — method detached from object | Not caught by default | Arrow function field or `bind` in constructor |
| Nested regular function inside a method | Regular function creates its own `this` binding | `noImplicitThis` (part of `strict`) flags `this` as implicit `any` | Replace nested function with an arrow function |
| Using `this` before `super()` in a subclass constructor | Parent class has not yet initialised the instance | Hard compile-time error in TypeScript | Call `super()` first |
| Expecting static methods to be fixed to the defining class | Static `this` is dynamic — refers to the class it is called on | No error; intentional behaviour | Use intentionally for factory patterns; document the expectation |

TypeScript does not solve the problem at the language level, but it gives you tools — the `this` parameter, `ThisType<T>`, and strict mode — to catch misuse early. Understanding where `this` is set, and applying the patterns shown above, is all you need to write TypeScript that behaves exactly as you expect.