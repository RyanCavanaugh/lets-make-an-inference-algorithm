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
theme: dracula
---

# Let's Make an Inference Algorithm

### Ryan Cavanaugh

Principal Software Engineeering Lead

TypeScript compiler team at Microsoft

-----------

# Goals for this Talk

<v-clicks>

 * Demystify TypeScript's inference process
 * Demonstrate how use cases drive language development
 * Discuss trade-offs

</v-clicks>

--------------

# Ground Rules

<v-clicks>

 * None of these examples are tricky on purpose!
 * Let's pretend soundness
 * No literal types

</v-clicks>

-----------

# Terminology

```ts{1|3|3-6}
function f<T>(arg1: T, arg2: T): T;

const result = f("hello", "world");
//             ^ what type is T in this call?
//    ^ what type is result?
```

------------

# Getting Started

```ts {1|3|5}
function f<T>(x: T): T;

let n: number = 1;

let x = f<__?__>(n);
```

<v-clicks depth="2">

* `any`
* `unknown`
* `number | string | boolean`
* `number`
   * ⭐
</v-clicks>

------------

# Algorithm

```ts
function f<T>(x: T): T;

let n: number = 1;

let x = f(n);
```

<v-clicks>

 * `x: T`
 * `n` passed to `x`
 * `n` is `number`
 * ∴ `T` is `number`

</v-clicks>

------------

# Algorithm

<v-clicks depth="2">

 1. Look at the argument type corresponding to the parameter that uses the type parameter
    * 🤔 what if there isn't one?

</v-clicks>


------------

# Wrapped Type Parameters

```ts{1|3-5|7|all}
type Box<T> = { readonly value: T };

function unbox<T>(b: Box<T>): T {
  return b.value;
}

const n = unbox({ value: 42 });
```

<v-clicks>
 
 * `b` is `{ value:   T    }`
 * `x` is `{ value: number }`
 * ∴ `T` is `number`

</v-clicks>

------------

# Algorithm

 1. At each parameter position, match occurrences of `T` in the argument types
 2. Set `T` according to those matches

------------

# Algorithm

<ol>
<li>Collect <i>candidates</i></li>
<li>Set <code>T</code> to a candidate <span v-click>← "a" candidate? 🤔</span></li>
</ol>

------------

# Multiple Candidates: Round 1

```ts{1|2-4|6|all}
type Box<T> = { readonly value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

<v-clicks>

 * `unknown`
 * `string | number`
 * `number` + an error on `"hello"`

</v-clicks>

<!-- unknown seems bad just on its face -->
<!-- What should happen? We need some principles -->

---
layout: center
---

# Generics should usually mimic what would happen if you inlined the call

<!-- This isn't always correct, but is a good starting point -->

--------------

# Inlined Call

```ts{1-4|4-7}
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}
const b = boxHasValue({ value: 42 }, "hello");

// Type error, can't compare string to number
const b = { value: 42 }.value === "hello"
```

<!-- If the inlined version errors, probably the call should error too -->

------------

# Multiple Candidates

```ts
type Box<T> = { readonly value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue({ value: 42 }, "hello");
```

<v-clicks>

 * `unknown` ← TypeScript 1.5 behavior
 * `string | number` ← maybe?
 * `number` + an error on `"hello"` ← TypeScript 1.6+ behavior

</v-clicks>

---------------

# Why Not Union?

```ts
type Box<T> = { readonly value: T };
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

```ts{3}
type Box<T> = { readonly value: T };

function boxHasValue<T, U>(box: Box<T>, value: U): boolean {
  return box.value === value;
}
```

<!-- Multiple type parameters should exist for a reason -->
<!-- especially when the type doesn't appear in the output -->

--------------

# Why Not Union?

```ts{3}
type Box<T> = { readonly value: T };

function boxHasValue(box: Box<unknown>, value: unknown): boolean {
  return box.value === value;
}
```

<!-- Multiple type parameters should exist for a reason -->
<!-- especially when the type doesn't appear in the output -->

--------------

# Algorithm

<v-clicks>

 1. Collect candidates
 2. Pick the first candidate
 3. Process the call (possibly erroring)

</v-clicks>

--------------

# Algorithm

```ts
type Box<T> = { readonly value: T };
function boxHasValue<T>(box: Box<T>, value: T): boolean {
  return box.value === value;
}

const b = boxHasValue<number>({ value: 42 }, "hello");
//                                           ~~~~~~~
//               Can't convert string to number
```

--------------

# Algorithm

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
* ∴ `T` = `string`
* ∴ Error on `(string | number)[]`

</v-clicks>

--------------

# Algorithm

 1. Collect candidates
 2. Pick the *best* candidate <span v-click>← "best" ? 🤔</span>
 3. Process the call

--------------

# Best Common Supertype

The largest type that contains all the other types

<v-clicks depth="2">

 * `string`, `number`, `string | number | boolean`
    * `string | number | boolean`
 * `Cat`, `Fish`, `Mammal`, `Animal`
    * `Animal`
 * `string`, `number`
    * (none)

</v-clicks>

--------------

# Multiple Candidates: Round 2

```ts
function find<T>(needle: T, haystack: readonly T[]): T;

find("hello", [1, 2, 3, "hello"]);
```

<v-clicks>

* Candidate 1: `string`
* Candidate 2: `string | number`
* ∴ `T` = `string | number`

</v-clicks>

---------------

# Multiple Candidates: Round 2

```ts
function find<T>(needle: T, haystack: readonly T[]): T;

find("hello", [1, 2, 3]);
```

<v-clicks>

* Candidate 1: `string`
* Candidate 2: `number`
* ∴ `T` = `string` (arbitrary)
* ∴ Error on `[1, 2, 3]`

</v-clicks>

--------------

# Algorithm

 1. Collect candidates
 2. Pick common supertype <span v-click>← 🤔?</span>
 3. Process the call

--------------

# Candidate Collection Challenge

```ts{1|3|5|all}
type Box<T> = { readonly value: T };

function unboxIfBox<T>(box: T | Box<T>): T;

const x = unboxIfBox({ value: 42 });
```

<v-clicks>

 * Candidate: `Box<number>`
 * Candidate: `number`
 * ∴ `x`: `Box<number>`
 * `Box<number>` 🤔⁉️

</v-clicks>

---
layout: center
---

# Observation: Deeper-nested occurrences of `T` are "better"

--------------

# Algorithm

 1. Collect candidates, noting their variance **and priority**
 2. Pick the best candidate, **preferring higher-priority inferences**
 3. Process the call

----------

# Inference Priority

```ts
type Box<T> = { value: T };

function unboxIfBox<T>(box: T | Box<T>): T;

const x = unboxIfBox({ value: 42 });
```

<v-clicks>

 * Candidate: `Box<number>` (lower-priority)
 * Candidate: `number` (higher-priority)
 * ∴ `x`: `number`
 * 🥰

</v-clicks>

--------------

# Nonsense Inference

```ts{1-4|5|7|all}
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
<li>∴ Best supertype: <code>string | number</code></li>
<li><code>string</code> 🤔⁉️</li>
</ul>
</v-clicks>

--------------

# Nonsense Inference

```ts{1-5|6|8|all}
function find<T>(
  arr1: readonly T[],
  arr2: readonly T[],
  func: (arg: T) => boolean
): T;
function isNeat(snb: string | number | boolean): boolean;

const x = find([1, 2, "a", "b"], [1, 2, 3], isNeat);
```

--------------

# Variance-based Bounds

```ts
function find<T>(arr1: T[], arr2: T[], func: (arg: T) => boolean): T;
function isNeat(snb: string | number | boolean): boolean;
const x = find([1, 2, "a", "b"], [1, 2, 3], isNeat);
```

| | | | | |
|--|--|--|--|--|
|<span v-click="1">`arg`</span>|<span v-click="2">`string \| number \| boolean`</span>|<span v-click="3">input</span>|<span v-click="10">upper bound</span>|
|<span v-click="4">`arr1`</span>|<span v-click="5">`string \| number`</span>|<span v-click="6">output</span>|<span v-click="11">best output</span><span v-click="13">⭐</span>|
|<span v-click="7">`arr2`</span>|<span v-click="8">`number`</span>|<span v-click="9">output</span>|<span v-click="12">other output</span>|

--------------

# Variance-based Bounds

```ts{3|all}
function find<T>(arr1: T[], arr2: T[], func: (arg: T) => boolean): T;

function isNeat(n: number): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
```

<v-clicks>

 * `string | number` ← best output
 * `number` ← upper bound from input ⭐
 * `number` ← lower output

</v-clicks>

-------------

# Variance-based Bounds

```ts
function find<T>(arr1: T[], arr2: T[], func: (arg: T) => boolean): T;

function isNeat(n: number): boolean;

const x = find([1, 2, 3], [1, 2, "a", "b"], isNeat);
//                               ~~~  ~~~
```

--------------

# Algorithm

 1. Collect candidates, noting their variance
 2. Pick the best candidate, bounded in both directions
 3. Process the call

------------

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

# Algorithm

<v-clicks depth="2">

 1. Collect candidates from non-context-sensitive expressions
 1. Pick the best candidate for type parameters with at least one candidate
 1. Repeat the process for context-sensitive expressions
 1. Process the call

</v-clicks>

--------------

# Algorithm

```ts
function exec<T, U>(
    consumer: (arg: T) => U,
    producer: () => T
): U;

const n = exec<string, number>(
    arg => arg.length,
    () => "hello"
);
```

<v-clicks>

 * `T`: `string`
 * `U`: `number`
 * 🥰

</v-clicks>

--------------

# Summary

 * Inference is not that scary!
 * Work from motivating examples
