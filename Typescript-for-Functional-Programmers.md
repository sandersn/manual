# Typescript for Functional Programmers

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

This introduction does not cover object-oriented programming. In
practise, object-oriented programs in Typescript are similar to those
in other popular languages with OO features.

# Prerequisites

In this introduction, I assume you know the following:

- How to program in Javascript, the good parts.
- Type syntax of a C-descended language.

If you need to learn the good parts of Javascript, read
[JavaScript: The Good Parts](http://shop.oreilly.com/product/9780596517748.do).
You may be able to skip the book if you know how to write programs in
a call-by-value lexically scoped language with lots of mutability and
not much else.
[R<sup>4</sup>RS Scheme](https://people.csail.mit.edu/jaffer/r4rs.pdf) is a good example.

[The C++ Programming Language](http://www.stroustrup.com/4th.html) is
a good place to learn about C-style type syntax. Unlike C++, Typescript uses postfix types, like so: `x: string` instead
of `string x`.

# Concepts not in Haskell

## Built-in types

Javascript defines 7 built-in types:

Type        | Explanation
------------|-----------
`Number`    | a double-precision IEEE 754 floating point.
`String`    | an immutable 16-bit string.
`Boolean`   | `true` and `false`.
`Symbol`    | a unique value usually used as a key.
`Null`      | equivalent to the unit type.
`Undefined` | also equivalent to the unit type.
`Object`    | similar to records.

[See the MDN page for more detail](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures).

Typescript has corresponding primitive types for the built-in types:

* `number`
* `string`
* `boolean`
* `symbol`
* `null`
* `undefined`
* `object`

Notes:

`null` and `undefined` are semantically different. They're both unit
types with a single value each, `null` and `undefined`, but the
runtime never produces `null`. It is reserved for authors to use.
Despite this, `undefined` is frequently used by authors instead.

### Other important types

Type        | Explanation
------------|-----------
`unknown` | the top type.
`never` | the bottom type.
`{}` | the smallest non-null, non-undefined type.
object literal | eg `{ property: Type }`
`void` | a subtype of `undefined` intended for use as a return type.
`T[]` | mutable arrays, also written `Array<T>`
`[T,T]` | tuples, which are fixed-length but mutable
`Function` | all functions, even ones produces from `eval`

Notes:

1. Function syntax includes parameter names. This is pretty hard to get used to!

    ```ts
    let fst: (a: any, d: any) => any = (a,d) => a;
    // or more precisely:
    let snd: <T, U>(a: T, d: U) => U = (a,d) => d.
    ```

2. Object literal type syntax closely mirrors object literal value syntax:

    ```ts
    let o: { n: number, xs: object[] } = { n: 1, xs: [] }
    ```

3. Confusingly, `{}` is the supertype of not only `object`, but
everything except `null` and `undefined`.

4. This means that the top type, `unknown`, is almost the same as
`{} | null | undefined`.

5. Even more confusingly, any object type with a property is a subtype of `object`, which is a subtype of `{}`.

6. `T[]` is a subtype of `Array` for any type `T`.

7. `[T, T]` is a subtype of `Array`, as well as a subtype of `T[]`. This is different than Haskell, where tuples are not related to lists.

8. `(t: T) => U` is a subtype of `Function`. Do not use `Function`.

### Apparent/boxed types

Javascript has boxed equivalents of primitive types that contain the
methods that programmers associate with those types. Typescript
reflects this with, for example, the difference between the primitive
type `number` and the boxed type `Number`. The boxed types are rarely
needed, since their methods return primitives.

```ts
1..toExponential()
// equivalent to
Number.prototype.toExponential.call(1)
```

Note that calling methods on numeric literals requires an additional
`.` to aid the parser.

## Gradual typing

Typescript uses the type `any` whenever it can't tell what the type of
an expression should be. Compared to `Dynamic`, calling `any` a type
is an overstatement. It just turns off the type checker
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
programmers, although Haskell and most MLs are not
structurally typed. Its basic form is pretty simple:

```ts
let o: { x: string } = { x: "hi", extra: 1 }; // ok
let o2: { x: string } = o; // ok
```

Here, the object literal `{ x: "hi", extra: 1 }` constructs a matching
literal type `{ x: string, extra: number }`. That
type is assignable to `{ x: string }` since
it has all the required properties and those properties have
assignable types. The extra property doesn't prevent assignment.

This is true for named types as well; for assignability purposes
there's no difference between the type alias `One` and the interface
type `Two` below. They both have a property `p: string`. (The behaviour with
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

## Unions

In Typescript, union types are untagged. In other words, they do not
generate discriminated unions like `data` does in Haskell. However,
you can often discriminate types in a union using built-in tags or
other properties.

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

`string`, `Array` and `Function` have built-in type predicates,
conveniently leaving the object type for the `else` branch. It is
possible, however, to generate unions that are difficult to
differentiate at runtime. For new code, it's best to build only
discriminated unions.

The following types have built-in predicates:

Type      | Predicate
----------|-----------
string    | `typeof s === "string"`
number    | `typeof n === "number"`
bigint    | `typeof m === "bigint"`
boolean   | `typeof b === "boolean"`
symbol    | `typeof g === "symbol"`
undefined | `typeof undefined === "undefined"`
function  | `typeof f === "function"`
array     | `Array.isArray(a)`
object    | `typeof o === "object"`

Note that functions and arrays are objects at runtime, but have their
own predicates.

### Intersections

In addition to unions, Typescript also has intersections:

```ts
type Combined = { a: number } & { b: string }
type Conflicting = { a: number } & { a: string }
```

`Combined` has two properties, `a` and `b`, just as if they had been
written as one object literal type. Intersection and union are
recursive in case of conflicts, so `Conflicting.a: number & string`.

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
to `string`. This happens when using mutability, which can hamper some
uses of mutable variables:

```ts
let s = "right";
pad("hi", 10, s); // error: 'string' is not assignable to '"left" | "right"'
```

Here's how the error happens:

* `"right": "right"`
* `s: string` because `"right"` widens to `string` on assignment to a mutable variable.
* `string` is not assignable to `"left" | "right"`

You can work around this with a type annotation for `s`, but that
in turn prevents assignments to `s` of variables that are not of type
`"left" | "right"`.

```ts
let s: "left" | "right" = "right";
pad("hi", 10, s);
```

## Ambient declarations

It may be necessary to declare that a Javascript value exists, even
though it is not part of the compilation. These declarations are called
"ambient declarations". They are particularly common when modelling
global code in the browser:

```ts
interface JQuery {
    // types here!
}
declare var $: JQuery;
```

But they can also be used to declare an entire module:

```ts
declare module "jquery" {
  interface JQuery {
      // types here!
  }
  declare var $: JQuery;
  export = $;
}
```

For an entire module, however, the usual Typescript solution is to
create a separate file with the extension `.d.ts`:

```ts
// @Filename: jquery.d.ts
interface JQuery {
    // types here!
}
declare var $: JQuery;
export = $;
```

A `.d.ts` will be used in place of a Javascript file, or even
Typescript file, if put in the correct location. See the section on
Declaration Files for more information.

## Merging

Typescript tries to provide types for Javascript values, but,
especially early in its life, the number of type constructors it had
was limited, and usually based on OO. To increase the expressivity of
this system, Typescript merges structures of different kinds that have
the same name. You can combine structures to represent a
single Javascript value with multiple Typescript declarations.

For example, you can represent a function that also has properties, or
nested types:

```ts
function pad(s: string, n: number, char: string, direction: pad.Direction): string {
}
namespace pad {
    export type Direction = "left" | "right";
    export const space = " ";
    export const tab = "\t";
    export const dot = ".";
}
pad('hi', 10, pad.dot, "left");
```

You can merge any two things that do not collide. This works across
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

# Concepts similar to Haskell

## Contextual typing

Typescript has some obvious places where it can infer types, like
variable declarations:

```ts
let s = "I'm a string!"
```

But it also infers types in a few other places that you may not expect
if you've worked with other C-syntax languages:

```ts
declare function map<T, U>(f: (t: T) => U, ts: T[]): U[];
let sns = map(n => n.toString(), [1,2,3]);
```

Here, `n: number` in this example also, despite the fact that `T` and `U`
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
`number`. And it can infer return types from context:

```ts
declare function run<T>(thunk: (t: T) => void): T
let i: { inference: string } = run(o => { o.inference = 'INSERT STATE HERE' });
```

The type of `o` is determined to be `{ inference: string }` because

1. Declaration initialisers are contextually typed by the
declaration's type: `{ inference: string }`.
2. The return type of a call uses the contextual type for inferences,
so the compiler infers that `T={ inference: string }`.
3. Arrow functions use the contextual type to type their parameters,
so the compiler gives `o: { inference: string }`.

And it does so while you are typing, so that after typing `o.`, you
get completions for the property `inference`, along with any other
properties you'd have in a real program.
Altogether, this feature can make Typescript's inference look a bit
like a unifying type inference engine, but it is not.

## Type aliases

Type aliases are mere aliases, just like `type` in Haskell. The
compiler will attempt to use the alias name wherever it was used in
the source code, but does not always succeed.

```ts
type Size = [number, number];
let x: Size = [101.1, 999.9];
```

The closest equivalent to `newtype` is a *tagged intersection*:

```ts
type FString = string & { __compileTimeOnly: any }
```

An `FString` is just like a normal string, except that the compiler
thinks it has a property named `__compileTimeOnly` that doesn't
actually exist. This means that `FString` can still be assigned to
`string`, but not the other way round.

## Discriminated Unions

The closest equivalent to `data` is a union of types with discriminant
properties, normally called discriminated unions in Typescript:

```ts
type Shape =
  | { kind: "circle", radius: number }
  | { kind: "square", x: number }
  | { kind: "triangle", x: number, y: number }
```

Unlike Haskell, the tag, or discriminant, is just a property in each
object type. Each variant has an identical property with a different
unit type. This is still a normal union type; the leading `|` is
an optional part of the union type syntax. You can discriminate the
members of the union using normal Javascript code:

```ts
function area(s: Shape) {
    if (s.kind === "circle") {
        return Math.PI * s.radius * s.radius;
    }
    else if (s.kind === "square") {
        return s.x * s.x;
    }
    else {
        return s.x * s.y / 2;
    }
}
```

Note that the return type is `area` is inferred to be `number` because
Typescript knows the function is total. If some variant is not
covered, the return type of `area` will be `number | undefined` instead.

Also unlike Haskell, common properties show up in any union, so you
can usefully discriminate multiple members of the union:

```ts
function height(s: Shape) {
    if (s.kind === "circle") {
        return 2 * s.radius;
    }
    else {
        // s.kind: "square" | "triangle"
        return s.x;
    }
}
```

## Type Parameters

Like most C-descended languages, Typescript requires declaration of
type parameters:

```ts
function liftArray<T>(t: T): Array<T> {
    return [t];
}
```

There is no case requirement, but the type parameters are conventionally
single uppercase letters. Type parameters can also be constrained to a
type, which behaves a bit like type class constraints:

```ts
function firstish<T extends { length: number }>(t1: T, t2: T): T {
    return t1.length > t2.length ? t1 : t2;
}
```

Typescript can usually infer type arguments from a call based on the
type of the arguments, so type arguments are usually not needed.

Because Typescript is structural, it doesn't need type parameters as
much as nominal systems. Specifically, they are not needed to make a
function polymorphic. Type parameters should only be used to
*propagate* type information, such as constraining parameters to be
the same type:

```ts
function length<T extends ArrayLike<unknown>>(t: T): number {
}

function length(t: ArrayLike<unknown>): number {
}
```

In the first `length`, T is not necessary; notice that it's only
referenced once, so it's not being used to constrain the type of the
return value or other parameters.

Typescript does not have higher kinded types, so the following is not legal:

```ts
function length<T extends ArrayLike<unknown>, U>(m: T<U>) {
}
```

## Module system

Javascript's modern module syntax is a bit like Haskell's, except that
any file with `import` or `export` is implicitly a module:

```ts
import { value, Type } from 'npm-package'
import { other, Types } from './local-package'
import * as prefix from '../lib/third-package'
```

You can also import commonjs modules &mdash; modules written using node.js'
module system:

```ts
import f = require('single-function-package');
```

You can export with an export list:

```ts
export { f }

function f() { return g() }
function g() { } // g is not exported
```

Or by marking each export individually:

```ts
export function f { return g() }
function g() { }
```

The latter style is more common but both are allowed, even in the same
file.

## `readonly` and `const`

In Javascript, mutability is the default, although it allows variable
declarations with `const` to declare that the *reference* is
immutable. The referent is still mutable:

```js
const a = [1,2,3];
a.push(102); // ):
a[0] = 101; // D:
```

Typescript additionally has a `readonly` modifier for properties.

```ts
interface Rx {
    readonly x: number;
}
let rx: Rx = { x: 1 };
rx.x = 12; // error
```

It also ships with a mapped type `Readonly<T>` that makes
all properties `readonly`:


```ts
interface X {
    x: number;
}
let rx: Readonly<X> = { x: 1 };
rx.x = 12; // error
```

And it has a specific `ReadonlyArray<T>` type that removes
side-affecting methods and prevents writing to indices of the array,
as well as special syntax for this type:

```ts
let a: ReadonlyArray<number> = [1, 2, 3];
let b: readonly number[] = [1, 2, 3];
a.push(102); // error
b[0] = 101; // error
```

You can also use a const-assertion, which operates on arrays and
object literals:

```ts
let a = [1, 2, 3] as const;
a.push(102); // error
a[0] = 101; // error
```

However, none of these options are the default, so they are not
consistently used in Typescript code.

# Further reading

TODO: Put links to the rest of the handbook here.

* Mapped types
* Index types and keyof types
* Conditional types
