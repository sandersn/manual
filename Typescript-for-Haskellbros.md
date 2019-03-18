# Typescript for Haskell programmers

Typescript began its life as an attempt to bring traditional object-oriented types
to Javascript so that the programmers at Microsoft could bring
traditional object-oriented programs to the web. As it has developed, Typescript's type
system has evolved to model code written by native Javascripters. The
resulting system is powerful, interesting and messy.

This introduction is designed for working Haskell or ML programmers
who want to learn Typescript. It describes how the type system of
Typescript differs from Haskell's type system.

# Things you should know

In this introduction, I assume you know the following:

- How to program in Javascript, the good parts.
- Type syntax of a C-descended language, including the syntax for type parameters.
- Semantics of a mainstream object-oriented type system.

If you need to learn the good parts of Javascript, read
[JavaScript: The Good Parts](http://shop.oreilly.com/product/9780596517748.do).
You may be able to skip the book if you know how to write programs in
a call-by-value lexically scoped language with lots of mutability and
not much else.

I guess
[The C++ Programming Language](http://www.stroustrup.com/4th.html) is
a good place to learn about C-style type syntax. Unlike earlier C-like
syntaxes, Typescript uses postfix types, like so: `x: string` instead
of `string x`.

I don't know a single place to learn the mushy concept of "mainstream
object-oriented type system", so just read section III of
[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/).

# Things that might be new

## Built-in types

Javascript defines 7 built-in types:

* `Number` - a double-precision IEEE 754 floating point.
* `String` - an immutable 16-bit string.
* `Boolean` - `true` and `false`.
* `Symbol` - a unique value usually used as a key.
* `Null` - equivalent to the unit type.
* `Undefined` - also equivalent to the unit type.
* `Object` - similar to records.

[See the MDN page for more detail](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures).

Notes:

1. I'm not joking. The only built-in numbers are floating points.
2. `Symbol('foo') !== Symbol('foo')`, so symbols are not like Lisp or Ruby atoms.
3. You can think of Javascript as a hasty copy of Scheme and you'd be nearly right.
4. Except for the inanely detailed implementation of object orientation, which you should avoid.

Typescript has corresponding primitive types for the built-in types:

* `number`
* `string`
* `boolean`
* `symbol`
* `null`
* `undefined`
* `object`

Notes:

`null` and `undefined` are semantically different. They're both
unit types with a single value each, `null` and `undefined`, but the
runtime never produces `null`. It is reserved for authors to use. As a
consequence, nobody ever uses it. `undefined` is considerably more
popular.

### Other important types

* `unknown` - the top type.
* `never` - the bottom type.
* `void` - a subtype of `undefined` intended for use by C programmers as a return type.
* `{}` - the smallest non-null, non-undefined type.
* object literal type - eg `{ property: Type }`
* `Array` - mutable arrays
* `tuples` - `[number, number]` - a subtype of arrays.
* `Function` - all functions, even dynamically evaluated ones

Notes:

1. Function syntax includes parameter names. This is pretty weird!

    ```ts
    let fst: (a: any, d: any) => any = (a,d) => a;
    // or more accurately:
    let snd: <T, U>(a: T, d: U) => U` = (a,d) => d.
    ```

2. Object type syntax closely mirrors value syntax:

    ```ts
    let o: { n: number, xs: object[] } = { n: 1, xs: [] }
    ```

    Remember that objects are mutable!

    ```ts
    let o: { a: readonly number, xs: readonly object[] } = { n: 1, xs: [] }
    ```

3. Confusingly, `{}` is the supertype of not only `object`, but also
`number`, `string` etc. Actually, everything except `null` and
`undefined`.
4. This means that the top type, `unknown` is almost the same as
`{} | null | undefined`. More on that later.
5. `T[]` is a subtype of `Array`.
6. `[T, T]` is a subtype of `Array`, as well as a subtype of `T[]`. This is different, and much less safe, than Haskell.
7. `(t: T) => U` is a subtype of `Function`.

### Apparent/boxed types

Javascript has boxed equivalents of primitive types that contain the methods that programmers associate with those types. Typescript reflects this with, for example, the difference between the primitive type `number` and the boxed type `Number`.

## Gradual typing

Typescript uses the type `any` whenever it can't tell what the type of an expression should be. Compared to `Dynamic`, calling `any` a type is an overstatement. It mostly just turns off the type checker wherever it appears. For example, you can push anything into an `any[]` without marking it in any way:

```ts
// with "noImplicitAny": false in tsconfig.json, anys: any[]
const anys = [];
anys.push(1);
anys.push("oh no");
anys.push({ anything: "goes" });
```

And you can use an expression of type `any` anywhere:

```ts
anys.map(anys[1]); // oh no, "oh no" is not a function
```

It is contagious, too &mdash; if you initialise a variable with an expression of type `any`, the variable has type `any` too. 

```ts
let sepsis = anys[0] + anys[1]; // this could mean anything
```

To get an error when Typescript produces an `any`, use `"noImplicitAny": true`, or `"strict": true`.

## Structural typing

Structural typing is 

### built-in relations

* undefined, void, [missing]
* object, {}, { a: number }

## Unit types
## merging

And now for something completely different.
The Typescript binder

# Things that are like Haskell/ML

But different. Usually worse. 

- contextual typing
  - is a *little* bit like HM
- untagged unions
- discriminated unions
- type parameters
  - constraints are a *little* bit like typeclasses
- type aliases
- tagged [unit] types
  - note which types are irregular (null and function, to wit)
- ad-hoc replacements for The Popular Monads
- modules
- readonly and const
  - You Will Be Sad after this

# Advanced types that are Javascript's fault

- declare/ambient syntax
- Mapped types

You know what? This section is so tiring. Go read about these types on their own pages.

# Advanced types that are not in Typescript

- Pattern matching
- Special syntax for the Maybe monad
- Special syntax for the List monad
- Lists at all
- Higher-kinded types

So don't try to write any monads!
