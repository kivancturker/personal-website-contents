---
title: "Mastering TypeScript Generics: Deep Dive"
date: "2026-07-10"
tags: ["typescript", "programming", "webdev"]
pinned: false
---

Generics are one of the most powerful features in modern TypeScript. They allow you to write reusable, type-safe code that acts as a blueprint for types, adapting seamlessly based on usage without falling back to the dreaded `any` type.

In this guide, we'll walk through some foundational concepts and quickly scale up to advanced utility pattern designs.

### The Basics: Generic Functions

A simple example of a generic function is one that returns an identity or processes an array element:

```typescript
function getFirstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

// Usage
const numberVal = getFirstElement([1, 2, 3]); // Type inferred as number
const stringVal = getFirstElement(["apple", "banana"]); // Type inferred as string
```

### Generic Constraints (`extends`)

Sometimes, you don't want a generic parameter to be *anything*. You want it to adhere to a specific structure. You can enforce this restriction using the `extends` keyword.

```typescript
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(arg: T): T {
  console.log(`Length: ${arg.length}`);
  return arg;
}

logLength("hello"); // Works (strings have .length)
logLength([1, 2, 3]); // Works (arrays have .length)
// logLength(123); // Error: Argument of type 'number' is not assignable to 'Lengthwise'
```

### Advanced: mapped and conditional generics

Suppose you want to write an updater function that takes an object, key, and value, and correctly types the updated property:

```typescript
function updateProperty<T, K extends keyof T>(obj: T, key: K, value: T[K]): T {
  return {
    ...obj,
    [key]: value
  };
}

const user = { id: 1, username: "dev_john", isActive: true };

// Safe and type-checked!
const updatedUser = updateProperty(user, "username", "john_doe");
// updateProperty(user, "isActive", "yes"); // Error: 'string' is not assignable to 'boolean'
```

Using these types of strict patterns ensures that code bases scaling to hundreds of files remain coherent, readable, and highly discoverable via IntelliSense!