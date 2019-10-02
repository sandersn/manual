The intent of this document is to teach you how to write JSDoc for
strict Typescript type checking. It assumes that you're writing JSDoc
in Javascript, either from scratch, or as part of a complete JSDoc
update. You'll learn how to add type annotations the recommended way,
starting with the most common types. We'll also cover the most common
problems and workarounds.

For this article, I assume that you know Javascript pretty well, and
know a little about Typescript and JSDoc.

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

If you need to give the type of a variable inside a function,
you can use the `@type` tag. Typescript can infer types from
initialisers, so you'll mostly need this when initialising something
complex that takes multiple steps, particularly inside a callback,
where control flow analysis can't reach:

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

Notice that Typescript supports two syntaxes for arrays: `T[]` and
`Array<T>`. You can use all the types from Typescript in
JSDoc, except for a few corners like conditional types. That's because
Typescript supports additional type syntax from the JSDoc
documentation generator and the Closure optimizing compiler, which
conflicts with Typescript's syntax for conditional types. This guide
only covers Typescript types, though.


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
  - importing types back and forth
- More advanced
  - `@extends`
  - `@constructor` - But prefer classes if you can
  - why use a d.ts file
  - when to switch to .ts
- Walkthrough 1: All-new JSDoc
- Walkthrough 2: Upgrading old JSDoc
- Walkthrough 3: Pure JSDoc
  - Please don't do this!
  - But it works.
  - Kind of ...
- Problems
  - Workarounds
    - Casts
- What works
  - Tags that you shouldn't use, but are supported
    - `@this` - Just use a class
    - `@callback` - Just use `@typedef` of a signature
    - `@enum`

  - Types, especially closure types
    You definitely shouldn't use these even when they're more convenient.
  - what additional types might you want


import * as stream from 'stream';

const x: stream.Readable = process.stdin;
