The intent of this document is to teach you how to write JSDoc for
strict Typescript type checking. It assumes that you're writing JSDoc
in Javascript, either from scratch, or as part of a complete JSDoc
update. You'll learn how to add type annotations the recommended way,
starting with the most common types. We'll also cover the most common
problems and workarounds.

For this article, I assume that you know Javascript pretty well, and
know a little about Typescript and JSDoc.

## Basics

The most common JSDoc tag you'll see is the `@param` tag, which lets
you specify the type of a function parameter:

```js
/**
 * @param {number} n - you can give a description here
 * @param {string} s
 */
function repeat(n, s) {
  var total = "";
  for (var i = 0; i < n; i++) {
    total += s
  }
  return total
}
```

Typescript will infer the return type of `repeat` for you if it can.
Recursive functions are the main thing it has trouble with. Either
way, you are free to specify a return type explicitly with `@returns`:

```js
/**
 * @param {number} n - you can give a description here
 * @param {string} s
 * @returns {string}
 */
function repeat(n, s) {
  var total = "";
  for (var i = 0; i < n; i++) {
    total += s
  }
  return total
}
```

If you need to give the type of a variable inside a function, you can
use the `@type` tag. Typescript can infer types from initialisers, so
you'll mostly need this for complex, multi-step initialisation,
particularly inside a callback, where control flow analysis can't
reach:

```ts
/**
 * @param {unknown[]} l
 */
function enumerate(l) {
  /** @type {Array<[unknown,number]>} */
  var pairs = []
  l.forEach((x,i) => pairs.push([x, i]))
  return pairs
}
```

Typescript supports two syntaxes for arrays: `T[]` and
`Array<T>`. You can use all the types from Typescript in
JSDoc, except for a few corners like conditional types. That's because
Typescript supports additional type syntax from the JSDoc
documentation generator and the Closure optimizing compiler, which
conflicts with Typescript's syntax for conditional types. This guide
only covers Typescript types, though.

For functions that use complex types like `Array<T>`, you will need to
specify the type parameter to get accurate types. For that, use the
`@template` tag:

```js
/**
 * @template T
 * @param {T[]} l
 */
function enumerate(l) {
  /** @type {Array<[T, number]>} */
  var pairs = []
  l.forEach((x, i) => pairs.push([x, i]))
  return pairs
}
```

You can also require a type parameter to extend some type, though this
is mainly useful for complex types.

```js
/**
 * @template T
 * @template {keyof T} K
 * @param {T} t
 * @param {K} k
 * @returns {T[K]}
 */
function logget(t, k) {
  const value = t[k]
  console.log(`requested key ${k}, got ${value}`)
  return value
}
```

This example has more lines of types than of code that runs! In cases
like this, sometimes the best option is to use simple but inaccurate
types, or to leave off types entirely.

## Intermediates

The preceding tags will be enough to get you started on type-checking
your Javascript code. However, there are a few techniques you need to
help you work the way the type checker expects. And there are a few
tags you'll need to fill in type information that Javascript leaves out.

### d.ts files

The most important technique is to use a .d.ts file for defining
non-class types. This puts all your types in one place, which is the
right default for all but the smallest and largest code bases. In a
.d.ts file, you can use the complete type syntax of Typescript, such
as conditional types. As a bonus, Typescript's syntax is more concise
than JSDoc.

Because of the way Typescript treats global types, you can make all
your types global:

```ts
type Predicate<T> = (x: T) => boolean
interface Config {
  /** must be less than 80 */
  spaces?: number
  alignment?: "left" | "right"
}
```

And use them, even in modules:

```js
/** @type {Predicate<Config>} */
export function isGood(config) {
  return (!config.spaces || config.spaces < 80)
}
```

### `@typedef`

The fifth important tag is `@typedef`, which lets you make type
aliases in Javascript. You should usually use a .d.ts file to declare
types but sometimes it is handy to have a local type alias. There are
several ways to write `@typedef`, but only the shortest is simple
enough for me to remember:

```js
/** @typedef {(x: unknown) => boolean} Predicate */
```

You can document the short `@typedef` after the type name:

```js
/** @typedef {string} Password - this string is secret */
```

For completeness, here is my preferred form of multi-line `@typedef`:

```js
/**
 * @typedef {Object} Config
 * @property {number} [spaces] - must be less than 80
 * @property {"left" | "right"} [alignment]
 */
```

But getting that syntax right is really quite difficult, so I would
avoid it.

TODO: Below here is extremely drafty writing:

### Casts

It is frustrating when Typescript gets confused about what type
something is. You can manually override the type using an inline
`@type` tag:

```js
somethingThatDemandsAConfig(/** @type {Config} */({}))
```

The syntax is a `@type` tag followed by parentheses surrounding the
thing you want to cast.

### `@extends`

When you extend a generic base class, the type checker will use `any`
for all of the type parameters since Javascript syntax doesn't have a
place for type arguments:

```js
class Matrix extends Array {
  // don't actually do this!
}
```

`@extends` lets you specify the type argument:

```js
/** @extends {Array<number>} */
class Matrix extends Array {
}
```

### `@constructor`

If your code still uses constructor functions instead of classes,
sometimes Typescript can't tell that a function is meant to be a
constructor function. In that case you can explicitly mark it as such:

```js
/** @constructor */
function Foo() { }
Foo.prototype.lazyInit = function(x) {
  this.value = x
}
```

## Other supported tags



- Basics
  - `@param`
  - `@returns`
  - `@type`
  - `@typedef`
  - `@template`
  - which are the supported tags
  - what are types
  - what are the primitive types
  - why declare types
- More advanced
  - .d.ts files
  - importing types back and forth
  - `@extends`
  - `@constructor` - But prefer classes if you can
  - why use a d.ts file
  - when to switch to .ts
  - import types, importing values vs types
  - casts
- What works
  - Tags that you shouldn't use, but are supported
    - `@this` - Just use a class
    - `@callback` - Just use `@typedef` of a signature
    - `@enum` - instead describe how *actually* to write an enum in JS.

  - Types, especially closure types
    You definitely shouldn't use these even when they're more convenient.
  - what additional types might you want

- Walkthrough 1: All-new JSDoc
- Walkthrough 2: Upgrading old JSDoc
