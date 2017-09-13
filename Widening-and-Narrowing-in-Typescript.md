# Widening and Narrowing in Typescript

Typescript has a number of related concepts in which a type gets
treated temporarily as a similar type. Some of these concepts are
internal-only. None of them are documented very well. For the internal
concepts, we expect nobody needs to know about them to use the
language. For the external concepts, we hope that they work well
enough that most people *still* don't need to think about them. This
document explains them all, aiming to help two audiences: (1) advanced
users of Typescript who *do* need to understand the quirks of the
language (2) contributors to the Typescript compiler.

The concepts covered in this document are as follows:

1. Widening: treat an internal type as a normal one.
2. Literal widening: treat a literal type as a primitive one.
3. Narrowing: remove constituents from a union type.
4. Type predicate narrowing: treat a type as a subclass.
5. Apparent type: treat a non-object type as an object type.

## Widening

Widening is the simplest operation of the bunch. The types `null` and
`undefined` are converted to `any`. This happens
recursively in object types, union types, and array types (including
tuples).

Why widening? Well, historically, `null` and `undefined` were internal
types that needed to be converted to `any` for downstream consumers
and for display. With `--strictNullChecks`, widening doesn't happen
any more. But without it, widening happens a lot, generally when obtaining
a type from another object. Here are some examples:

```ts
// @strict: false
let x = null;
```

Here, `null` has the type `null`, but `x` has the type `any` because
of widening on assignment. `undefined` works the same way. However,
with `--strict`, `null` is preserved, so no widening will happen.

## Literal widening

Literal widening is significantly more complex than "classic"
widening. Basically, when literal widening happens, a literal type
like `"foo"` or `SomeEnum.Member` gets treated as its base type:
`string` or `SomeEnum`, respectively. The places where literals widen,
however, cause the behaviour to be hard to understand. Literal
widening is described fully
[at the literal widening PR](https://github.com/Microsoft/TypeScript/pull/10676)
and
[its followup](https://github.com/Microsoft/TypeScript/pull/11126).

### When does literal widening happen?

There are two key points to understand about literal widening.

1. Literal widening only happens to literal types that originate from
expressions. These are called *fresh* literal types.
2. Literal widening happens whenever a fresh literal type reaches a
"mutable" location.

For example,

```ts
const one = 1; // 'one' has type: 1
let num = 1;   // 'num' has type: number
```

Let's break the first line down:

1. `1` has the fresh literal type `1`.
2. `1` is assigned to `const one`, so `one: 1`. But the type `1` is still
fresh! Remember that for later.

Meanwhile, on the second line:

1. `1` has the fresh literal type `1`.
2. `1` is assigned to `let num`, a mutable location, so `num: number`.

Here's where it gets confusing. Look at this:

```ts
const one = 1;
let wat = one; // 'wat' has type: number
```

The first two steps are the same as the first example. The third step

1. `1` has the fresh literal type `1`.
2. `1` is assigned to `const one`, so `one: 1`.
3. `one` is assigned to `wat`, a mutable location, so `wat: number`.

This is pretty confusing! The fresh literal type `1` makes its way
*through* the assignment to `one` down to the assignment to `wat`. But
if you think about it, this is what you want in a real program:

```ts
const start = 1001;
const max = 100000;
// many (thousands?) of lines later ...
for (let i = start; i < max; i = i + 1) {
  // did I just write a for loop?
  // is this a C program?
}
```

If the type of `i` were `1001` then you couldn't write a for loop based
on constants.

There are other places that widen besides assignment. Basically it's
anywhere that mutation could happen:

```ts
const nums = [1, 2, 3]; // 'nums' has type: number[]
nums[0] = 101; // because Javascript arrays are always mutable

const doom = { e: 1, m: 1 }
doom.e = 2 // Mutable objects! We're doomed!

// Dooomed!
// Doomed!
// -gasp- Dooooooooooooooooooooooooooooooooo-
```

### How does literal widening work?

TODO: This is just notes right now.

The core pieces of literal widening are `getBaseTypeOfLiteralType`, `getWidenedLiteralType` and
`getRegularTypeOfLiteralType`. These three functions are fairly simple:

`getBaseTypeOfLiteralType` converts string literal types to `string`,
number literal types to `number`, boolean literal types to `boolean`,
and literal enum members (which are usable as types) to the base enum
type. It also recurs through unions. `getWidenedLiteralType` is the
same code, except that it only widens if the type is marked with
`TypeFlags.FreshLiteral`.

`getRegularTypeOfLiteralType` gets the 'regular' version of a literal
type, which is one that will not widen to its primitive base type in `getWidenedLiteralType`.

* 'fresh' types are the ones that widen upon assignment to mutability)
* `getFreshTypeOfLiteralType` is called on numeric and string
  literals (plus unary prefix, but whatever)
* `getRegularTypeOfLiteralType` is called on type nodes, case clauses
* It's also called internally wherever we need type identity, which is
  a lot of places. This choice is somewhat arbitrary, except that
  `getRegularTypeOfLiteralType` is already called more places, so
  might as well keep fresh-type calls few.
* `getRegularTypeOfExpression` delegates to it. But it's only called
  for type-identity purposes too.
* `isRelatedTo` manually grabs the regularType of a fresh literal type
* `getWidenedLiteralType` is the one widening fresh literal types, and is called from...
  - JSX for ad-hoc reasons
  - type inference for complex reasons
  - `getContextuallyTypedParameterType`, during IIFE typings, because it's writable I guess?
  - `getReturnTypeFromBody`, when there is a contextual return type and the return type is a unit type
  - initializers that aren't const/readonly or a type assertion
* This is contrast to `getBaseTypeOfLiteralType`, which *always* widens and is called from:
  - doLiteralTypesHaveSameBaseType (`literalTypesWithSameBaseType`)
  - type inference, I think to find better inferences
  - `addEvolvingArrayElementType`, because it's by definition mutable
  - during flow analysis of compound assignments (like *= and +=)
  - and evolving assignments
  - in checkIdentifier, if the identifier is being assigned to
  - and in checkProperyAccessExpression
  - in checkAssertionWorker, maybe because the source is going to be cast to something else anyway?
  - checking >= and the other relational operators, as well as &&, so allowed comparisons aren't *too* strict
  - checking case clauses, for similar reasons


