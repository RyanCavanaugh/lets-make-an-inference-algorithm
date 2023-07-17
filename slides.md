---
theme: dracula
canvasWidth: 700
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
fonts:
  mono: 'IntelOne Mono'
transition: fade
title: Let's Make an Inference Algorithm
---

# Let's Make an Inference Algorithm

Ryan Cavanaugh
Principal Software Engineeering Lead
TypeScript compiler team at Microsoft

-----------

# The Problem

```ts
function f<T>(arg1: T, arg2: T): T;

const result = f("hello", "world");
//             ^ what type is T in this call?
//    ^ what type is result?
```

<!-- I will use a shorthand syntax -->

-----------

# Goals for this Talk

 * Demonstrate how use cases drive language development
 * Follow a simple model of TypeScript's inference process
 * Understand trade-offs

--------------

# Ground Rules

 * None of these examples are tricky on purpose!
 * Let's pretend soundness
 * No literal types

------------

# Getting Started

```ts
function id<T>(x: T): T;

let n: number = 1;

let x = id(n);
```

------------

# Getting Started

```ts
function id<T>(x: T): T;

let n: number = 1;

let x = id<__?__>(n);
```

 * `any`
 * `unknown`
 * `number | string | boolean`
 * `number`

------------

# Algorithm 1

 1. Look at the argument type corresponding to the generic parameter

------------

```ts
function id<T>(x: T): T;

let n: number = 1;

let x = id<number>(n);
```

------------

# Wrapped Type Parameters

```ts
type Box<T> = { value: T };

function unbox<T>(b: Box<T>): T {
  return b.value;
}

const n = unbox({ value: 42 });
```

------------

# Algorithm 2

 1. At each parameter position, match occurrences of `T` in the argument types
 2. Set `T` according to those matches

---------------

# Wrapped Type Parameters

```ts
type Box<T> = { value: T };

function unbox<T>(b: Box<T>): T {
  return b.value;
}

const n = unbox({ value: 42 });

// { value: number }
// { value:   T    }
// T = number
```

<!-- Here, we line up the occurrences of T with the manifest types -->

------------

# Algorithm 2 (Restated)

 1. Collect *candidates*
 2. Set `T` to a candidate

------------

# Algorithm 2

 1. Collect candidates
 2. Set `T` to a candidate ← "a" candidate? 🤔

------------

# Multiple Candidates: Round 1

```ts
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

------------

# Multiple Candidates: Round 1

```ts
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

 * `unknown`
 * `string | number`
 * `string` + an error on `"hello"`

<!-- unknown seems bad just on its face -->
<!-- What should happen? We need some principles -->

--------------

# Good Guideline

Generics should usually mimic what would happen if you inlined the call

<!-- This isn't always correct, but is a good starting point -->

--------------

# Good Guideline

```ts
const b = boxHasValue({ value: 42 }, "hello");
// Type error, can't compare string to number
const b = { value: 42 }.value === "hello"
```

<!-- If the inlined version errors, probably the call should error too -->

------------

# Multiple Candidates: Round 1 Observations

```ts
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

 * `unknown` ← TypeScript 1.5- behavior
 * `string | number` ← maybe...
 * `string` + an error on `arg2` ← TypeScript 1.6+ behavior

---------------

# Why Not Union?

```ts
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

<!-- string | number seems kinda reasonable -->
<!-- ... but then why did you write the function this way? -->

--------------

# Guiding Principle

> Developer intent exists and can be inferred from code structure


---------------

# Why Not Union?

```ts
type Box<T> = { value: T };
function boxHasValue<T, U>(box: Box<T>, value: U): boolean {
  return box.value === value;
}
```

<!-- Multiple type parameters should exist for a reason -->
<!-- especially when the type doesn't appear in the output -->

--------------

# Algorithm 3

 1. Collect candidates
 2. Pick the first candidate
 3. Process the call (possibly erroring)

--------------

# Algorithm 3

```ts
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue<number>({ value: 42 }, "hello");
//                                           ~~~~~~~
//               Can't convert string to number
```

--------------

# Algorithm 3

 1. Collect candidates
 2. Pick the first candidate 🤔
 3. Process the call

-------------

# Multiple Candidates: Round 2

```ts
function find<T>(needle: T, haystack: readonly T[]): T;
find("hello", [1, 2, 3, "hello"]);
```

* Candidate 1: `string`
* Candidate 2: `string | number`

--------------

# Algorithm 4

 1. Collect candidates
 2. Pick the *best* candidate
 3. Process the call

--------------

# Algorithm 4

 1. Collect candidates
 2. Pick the *best* candidate 🤔💭🏆
 3. Process the call

--------------

# Best Common Supertype

From the candidates, choose the type that is a supertype of all other types

 * `string`, `number`, `string | number | boolean`
    * `string | number | boolean`
 * `Cat`, `Fish`, `Mammal`, `Animal`
    * `Animal`
 * `string`, `number`
    * (none)

--------------

# Best Common Supertype Failure

```ts
function find<T>(needle: T, haystack: readonly T[]): T;
find("hello", [1, 2, 3]);
```

<!-- What to do when there are no failures? Press on -->

--------------

# Algorithm 4

 1. Collect candidates
 2. Pick the *best* candidate
   * A common supertype if it exists, otherwise arbitrary
 3. Process the call

--------------

# Algorithm 4

 1. Collect candidates
 2. Pick the *best* candidate
   * A common supertype 🤔💭 is this always right?
 3. Process the call

--------------

# Contravariant Inference

```ts
function find<T>(
  arr: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(sn: string | number): boolean;

const x = find([1, 2, 3], isNeat);
```

--------------

# Contravariant Inference

```ts
function find<T>(
  arr: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(sn: string | number): boolean;

const x = find([1, 2, 3], isNeat);
```

 * Candidate from `arr`: `number`
 * Candidate from `isNeat`: `string | number`
 * Best supertype: `string | number`

--------------

# Contravariant Inference

```ts
function find<T>(
  arr: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(sn: string | number): boolean;

const x = find([1, 2, 3], isNeat);
```

 * Candidate from `arr`: `number`
 * Candidate from `isNeat`: `string | number`
 * Best supertype: `string | number` 🤔

--------------

# Contravariant Inference

```ts
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(snb: string | number | boolean): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```

--------------

# Contravariant Inference

```ts
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(snb: string | number | boolean): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```

 * `string | number | boolean` (contravariant)
 * `string | number` (covariant)
 * `number` (covariant)

--------------

# Contravariant Inference

```ts
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(snb: string | number | boolean): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```
 * `string | number | boolean` ← contravariant upper bound
 * `string | number` ← highest covariant inference ⭐
 * `number` ← lower covariant inference
 
--------------

# Contravariant Inference

```ts
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(n: number): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```
 * `string | number` ← highest covariant inference
 * `number` ← contravariant upper bound ⭐
 * `number` ← lower covariant inference

--------------

# Algorithm 5

 1. Collect candidates, noting their variance
 2. Pick the best candidate, bounded in both directions
 3. Process the call

-------------

# Contravariant Inference

```ts
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(n: number): boolean;

const x = find<number>([1, 2, 3], [1, 2, "a", "b"], isNeat);
// Error                                 ~~~  ~~~
```

------------

# Candidate Collection Challenge

```ts
type Box<T> = { value: T };
function maybeUnbox<T>(box: T | Box<T>): T;

const x = maybeUnbox({ value: 42 });
```

------------

# Candidate Collection Challenge

```ts
type Box<T> = { value: T };
function maybeUnbox<T>(box: T | Box<T>): T;

const x = maybeUnbox({ value: 42 });
```

 * Candidate: `{ value: number }`
 * Candidate: `number`

------------

# Candidate Collection Challenge

```ts
type Box<T> = { value: T };
function maybeUnbox<T>(box: T | Box<T>): T;

const x = maybeUnbox({ value: 42 });
```

 * Candidate: `{ value: number }`
 * Candidate: `number`
 * `T` = `{ value: number }` 🤔

----------

# Inference Priority

```ts
type Box<T> = { value: T };
function maybeUnbox<T>(box: T | Box<T>): T;

const x = maybeUnbox({ value: 42 });
```

Observation: Deeper-nested occurrences of `T` are "better"


--------------

# Algorithm 6

 1. Collect candidates, noting their variance **and priority**
 2. Pick the best candidate, **preferring higher-priority inferences**
 3. Process the call

----------

# Context-Sensitive Expressions

```ts
declare function exec<T, U>(
    consumer: (arg: T) => U,
    producer: () => T
): U;

const n = exec(
    arg => arg.length,
    () => "hello"
);
```

--------------

# Context-Sensitive Expressions

```ts
declare function exec<T, U>(
    consumer: (arg: T) => U,
    producer: () => T
): U;

const n = exec(
    arg => arg.length,
    () => "hello"
);
```

 * Candidate for `T`: implicit `any`
 * Candidate for `U`: `any.length` -> `any`
 * Candidate for `T`: `string`

`U: any` 🤔

--------------

# Algorithm 7

 1. Collect candidates from non-context-sensitive expressions
 2. Pick the best candidate for type parameters with at least one candidate
 3. Repeat the process for context-sensitive expressions
 4. Process the call

--------------

# Algorithm 7

```ts
declare function exec<T, U>(
    consumer: (arg: T) => U,
    producer: () => T
): U;

const n = exec<string, number>(
    arg => arg.length,
    () => "hello"
);
```
