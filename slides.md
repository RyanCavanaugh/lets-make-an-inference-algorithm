---
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

### Ryan Cavanaugh

Principal Software Engineeering Lead

TypeScript compiler team at Microsoft

-----------

# Goals for this Talk

<v-clicks>

 * Demonstrate how use cases drive language development
 * Follow a simple model of TypeScript's inference process
 * Understand trade-offs

</v-clicks>

--------------

# Ground Rules

<v-clicks>

 * None of these examples are tricky on purpose!
 * Let's pretend soundness
 * No literal types

</v-clicks>

-----------

# The Problem

```ts{1|3|3-6}
function f<T>(arg1: T, arg2: T): T;

const result = f("hello", "world");
//             ^ what type is T in this call?
//    ^ what type is result?
```

------------

# Getting Started

```ts {1|3|5}
function id<T>(x: T): T;

let n: number = 1;

let x = id<__?__>(n);
```

<v-clicks>

* `any`
* `unknown`
* `number | string | boolean`
* `number`

</v-clicks>

------------

# Algorithm 1

 1. Look at the argument type corresponding to the generic parameter

------------

# Algorithm 1

```ts
function id<T>(x: T): T;

let n: number = 1;

let x = id<number>(n);
```

<v-clicks>

 * `x: T`
 * `n` passed to `x`
 * `n` is `number`
 * ∴ `T` is `number`

</v-clicks>

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

```ts{1|3-5|7|8|all}
type Box<T> = { value: T };

function unbox<T>(b: Box<T>): T {
  return b.value;
}

const x = { value: 42 };
const n = unbox(x);
```

<v-clicks>
 
 * `b` is `{ value:   T    }`
 * `x` is `{ value: number }`
 * ∴ `T` is `number`

</v-clicks>

<!-- Here, we line up the occurrences of T with the manifest types -->

------------

# Algorithm 2

<ol>
<li>Collect <i>candidates</i></li>
<li>Set <code>T</code> to a candidate <span v-click>← "a" candidate? 🤔</span></li>
</ol>

------------

# Multiple Candidates: Round 1

```ts{1|2-4|6|all}
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

<v-clicks>

 * `unknown`
 * `string | number`
 * `string` + an error on `"hello"`

</v-clicks>

<!-- unknown seems bad just on its face -->
<!-- What should happen? We need some principles -->

---
layout: center
---

# Generics should *usually* mimic what would happen if you inlined the call

<!-- This isn't always correct, but is a good starting point -->

--------------

# Inlined Call

```ts{1|3-4}
const b = boxHasValue({ value: 42 }, "hello");

// Type error, can't compare string to number
const b = { value: 42 }.value === "hello"
```

<!-- If the inlined version errors, probably the call should error too -->

------------

# Multiple Candidates

```ts
type Box<T> = { value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

<v-clicks>

 * `unknown` ← TypeScript 1.5 behavior
 * `string | number` ← maybe?
 * `string` + an error on `arg2` ← TypeScript 1.6+ behavior

</v-clicks>

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

---
layout: center
---

# Developer intent exists and can be inferred from code structure

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

# Why Not Union?

```ts
type Box<T> = { value: T };

function boxHasValue(box: Box<unknown>, value: unknown): boolean {
  return box.value === value;
}
```

<!-- Multiple type parameters should exist for a reason -->
<!-- especially when the type doesn't appear in the output -->

--------------

# Algorithm 3

<v-clicks>

 1. Collect candidates
 2. Pick the first candidate
 3. Process the call (possibly erroring)

</v-clicks>

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
 2. Pick the first candidate <span v-click>← "first"? 🤔</span>
 3. Process the call

-------------

# Multiple Candidates: Round 2

```ts{1|3}
function find<T>(needle: T, haystack: readonly T[]): T;

find("hello", [1, 2, 3, "hello"]);
```

<v-clicks>

* Candidate 1: `string`
* Candidate 2: `string | number`

</v-clicks>

--------------

# Algorithm 4

 1. Collect candidates
 2. Pick the *best* candidate <span v-click>← "best" ? 🤔</span>
 3. Process the call

--------------

# Best Common Supertype

From the candidates, choose the type that is a supertype of all other types

<v-clicks depth="2">

 * `string`, `number`, `string | number | boolean`
    * `string | number | boolean`
 * `Cat`, `Fish`, `Mammal`, `Animal`
    * `Animal`
 * `string`, `number`
    * (none)

</v-clicks>

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
 2. Pick common supertype if it exists <span v-click>← 🤔</span>
 3. Process the call

--------------

# Contravariant Inference

```ts{1-4|6|8|all}
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

<v-clicks>
<ul>
<li>Candidate from <code>arr</code>: <code>number</code></li>
<li>Candidate from <code>isNeat</code>: <code>string | number</code></li>
<li>Best supertype: <code>string | number</code> 🤔⁉️</li>
</ul>
</v-clicks>

--------------


# Contravariant Inference

```ts{1-5|7|9|all}
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(snb: string | number | boolean): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```

<v-clicks>

 * `string | number | boolean` (contravariant)
 * `string | number` (covariant)
 * `number` (covariant)

</v-clicks>

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

 * `string | number | boolean` <span v-click>← contravariant upper bound</span>
 * `string | number` <span v-click>← highest covariant inference ⭐</span>
 * `number` <span v-click>← lower covariant inference</span>
 
--------------

# Contravariant Inference

```ts{1-5|7|9|all}
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;

function isNeat(n: number): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```

<v-clicks>

 * `string | number` ← highest covariant inference
 * `number` ← contravariant upper bound ⭐
 * `number` ← lower covariant inference

</v-clicks>

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

--------------

# Algorithm 5

 1. Collect candidates, noting their variance
 2. Pick the best candidate, bounded in both directions
 3. Process the call

------------

# Candidate Collection Challenge

```ts{1,3,5,all}
type Box<T> = { value: T };

function maybeUnbox<T>(box: T | Box<T>): T;

const x = maybeUnbox({ value: 42 });
```

<v-clicks>

 * Candidate: `{ value: number }`
 * Candidate: `number`
 * `x`: `number | { value: number }`
 * 🤔⁉️

</v-clicks>

---
layout: center
---

# Observation: Deeper-nested occurrences of `T` are "better"

--------------

# Algorithm 6

 1. Collect candidates, noting their variance **and priority**
 2. Pick the best candidate, **preferring higher-priority inferences**
 3. Process the call

----------

# Inference Priority

```ts
type Box<T> = { value: T };

function maybeUnbox<T>(box: T | Box<T>): T;

const x = maybeUnbox({ value: 42 });
```

<v-clicks>

 * Candidate: `{ value: number }` (lower-priority)
 * Candidate: `number` (higher-priority)
 * `x`: `number`

</v-clicks>

----------

# Context-Sensitive Expressions

```ts{1-4|5-8|all}
function exec<T, U>(
    consumer: (arg: T) => U,
    producer: () => T
): U;
const n = exec(
    arg => arg.length,
    () => "hello"
);
```

<v-clicks>

 * Candidate for `T`: implicit `any`
 * Candidate for `U`: `any.length` -> `any`
 * Candidate for `T`: `string`
 * `U: any` 🤔

</v-clicks>

--------------

# Algorithm 7

<v-clicks depth="2">

 1. Collect candidates from non-context-sensitive expressions
 1. Pick the best candidate for type parameters with at least one candidate
 1. Repeat the process for context-sensitive expressions
 1. Process the call

</v-clicks>

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
