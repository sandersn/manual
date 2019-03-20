# Typescript for Haskell programmers

Typescript began its life as an attempt to bring traditional object-oriented types
to Javascript so that the programmers at Microsoft could bring
traditional object-oriented programs to the web. As it has developed, Typescript's type
system has evolved to model code written by native Javascripters. The
resulting system is powerful, interesting and messy.

This introduction is designed for working Haskell or ML programmers
who want to learn Typescript. It describes how the type system of
Typescript differs from Haskell's type system. It also describes
unique features of Typescript's type system that arise from its
modelling of Javascript code.

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

1. Function syntax includes parameter names. This is pretty hard to get used to!

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
5. Even more confusingly, any object type with a property is a subtype of `object`, which is a subtype of `{}`.
6. `T[]` is a subtype of `Array`.
7. `[T, T]` is a subtype of `Array`, as well as a subtype of `T[]`. This is different, and much less safe, than Haskell.
8. `(t: T) => U` is a subtype of `Function`.

### Apparent/boxed types

Javascript has boxed equivalents of primitive types that contain the
methods that programmers associate with those types. Typescript
reflects this with, for example, the difference between the primitive
type `number` and the boxed type `Number`.

## Gradual typing

Typescript uses the type `any` whenever it can't tell what the type of
an expression should be. Compared to `Dynamic`, calling `any` a type
is an overstatement. It mostly just turns off the type checker
wherever it appears. For example, you can push anything into an
`any[]` without marking it in any way:

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

It is contagious, too &mdash; if you initialise a variable with an
expression of type `any`, the variable has type `any` too.

```ts
let sepsis = anys[0] + anys[1]; // this could mean anything
```

To get an error when Typescript produces an `any`, use `"noImplicitAny": true`, or `"strict": true`.

## Structural typing

Structural typing is a familiar concept to most functional
programmers, although Haskell and most MLs are not actually
structurally typed. Its basic form is pretty simple:

```ts
let o: { x: string } = { x: "hi", extra: 1 }; // ok
let o2: { x: string } = o; // ok
```

That is, object literals construct a matching literal type. That
object literal type is assignable to some other object type as long as
it has all the required properties and those properties have
assignable types.

This is true for named types as well; for assignability purposes
there's no difference between the type alias `One` and the interface
type `Two`. They both have a property `p: string`. (The behaviour with
respect to recursive definitions and type parameters is slightly
different, however.)

```ts
type One = { p: string };
interface Two { p: string };
class Three { p = "Hello" };

let x: One = { p: 'hi' };
let two: Two = x;
two = new Three();
```

## Merging

And now for something completely different.

Typescript tries to provide types for Javascript values, but,
especially early in its life, the number of type constructors it had
was limited, and usually based on OO. To increase the expressivity of
this system, Typescript merges structures of different kinds that have
the same name. Programmers can combine structures to represent a
single Javascript value with multiple Typescript types.

For example, you can represent a function that also has properties, or
nested types:

```ts
function pad(s: string, n: number, char: string, direction: pad.Direction) {
}
namespace pad {
    export type Direction = "left" | "right";
    export const space = " ";
    export const tab = "\t";
    export const dot = ".";
}
pad('hi', 10, pad.dot, "left");
```

You merge any two things that will not collide. This works across
files too:

```ts
// @Filename: main.ts
interface Main {
    // lots of properties here
}

// @Filename: extra.ts
interface Main {
    extra: boolean;
}
```

## Unions

In Typescript, union types are untagged. In other words, they do not generate discriminated unions like `data` does in Haskell. However, you can often discriminate types in a union using built-in tags or other properties.

```ts
function lol(arg: string | string[] | () => string | { s: string }): string {
    // this is super common in Javascript
    if (typeof arg === 'string') {
        return commonCase(arg);
    }
    else if (Array.isArray(arg)) {
        return arg.map(commonCase).join(",");
    }
    else if (typeof arg === 'function') {
        return commonCase(arg());
    }
    else {
        return commonCase(arg.s);
    }

    function commonCase(s: string): string {
        // finally, just convert a string to another string
    }
}
```

`string`, `Array` and `Function` have built-in type predicates, conveniently leaving the object type for the `else` branch. It is possible, however, to generate unions that are difficult to differentiate at runtime. For new code, it's best to build only discriminated unions.


## Unit types

Unit types are subtypes of primitive types that contain exactly one
primitive value. For example, the string `"foo"` has the type
`"foo"`. Since Javascript has no built-in enums, it is common to use a set of
well-known strings instead. Unions of string literal types allow
Typescript to type this pattern:

```ts
declare function pad(s: string, n: number, direction: "left" | "right"): string;
pad("hi", 10, "left");
```
When needed, the compiler *widens* &mdash; converts to a
supertype &mdash; the unit type to the primitive type, such as `"foo"`
to `string`. However, it make some usages difficult:

```ts
let s = "right";
pad("hi", 10, s);
```

Here, `"right": "right"` but `"right"` widens to `string` because
`let` declares a mutable variable. This means that `s: string`, and
`string` is not assignable to `"left" | "right"` because it could have
some other type, like `"chips"`. You'll have to explicitly declare the
type of `s` to be `"left" | "right"`.

# Things that look like Haskell, but aren't

## Contextual typing

Typescript has some obvious places where it can infer types, like
variable declarations:

```ts
let s = "I'm a string!"
```

But it also infers types in a few other places that you may not expect
if you've worked with other C-syntax languages:

```ts
// functions
[1,2,3].map(n => n) // n: number
```

Here `n: number`, as determined by the type of `Array<number>.map`.
However, contextually typed lambdas can participate usefully in calls
where the type parameters must still be inferred:

```ts
declare function map<T, U>(f: (t: T) => U, ts: T[]): U[];
let sns = map(n => n.toString(), [1,2,3]);
```

`n: number` in this example also, despite the fact that `T` and `U`
have not been inferred before the call. In fact, after `[1,2,3]` has
been used to infer `T=number`, the return type of `n => n.toString()`
is used to infer `U=string`, causing `sns` to have the type
`string[]`.

Note that inference will work in any order, but intellisense will only
work left-to-right, so Typescript prefers to declare `map` with the
array first:

```ts
declare function map<T, U>(ts: T[], f: (t: T) => U): U[];
```

Contextual typing also works recursively through object literals, and
on unit types that would otherwise be inferred as `string` or
`number`. Altogether, this feature can make Typescript's inference
look a bit like unifying type inference engine, but it is not.

## Type aliases

Type aliases are mere aliases, just like `type` in Haskell, but unlike
`newtype` or `data`. The compiler will attempt to use the alias name
wherever it was used in the source code, but does not always succeed.

The closest equivalent to `newtype` is a *tagged intersection*:

```ts
type FString = string & { __compileTimeOnly: any }
```

An `FString` is just like a normal string, except that the compiler thinks it has a property named `__compileTimeOnly` that doesn't actually exist. This means that `FString` can still be assigned to `string`, but not the other way round.

The closest equivalent to `data` is a union of types with discriminant properties.

## Discriminated Unions


- type parameters
  - constraints are a *little* bit like typeclasses
- type aliases
- tagged [unit] types
  - note which types are irregular (null and function, to wit)
- ad-hoc replacements for The Popular Monads
- modules
- function types are named
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
