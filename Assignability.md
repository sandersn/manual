# Assignability

1. Basics
2. Structural assignability
3. Algebraic assignability
4. Hacks and tricks

## Basics

Assignability is, essentially, the function that determines whether
one variable can be assigned to another. It happens when the compiler checks
assignments and function calls, but also return statements. For
example:

```ts
var x: number;
var y: number;
x = y;

function f(s: string): number {
  return s.length;
}
f(false);
```

The first assignment is fine because both `x` and `y` have the type
`number`. The second assignability check is on the return
statement, which checks that string.length's type, `string`, is assignable to
`number`. The third check is that the type of false, `boolean`, is assignable to
`string`. Which it isn’t.

For a simple type system consisting only of the primitive types,
assignability would just be equality:

```ts
function isAssignableTo(source: Type, target: Type): boolean {
  return source === target;
}
```

For brevity's sake, I'll often use the operator ⟹ to represent
`isAssignableTo`, like this:

`number ⟹ number`.

In fact, one common bug in the compiler is to use equality where
assignability is the correct check. This is a bug because Typescript
supports many more types, and kinds of types, than just the primitive
types. Good examples include classes and interfaces, unions and
intersections, and literal types and tuples. Those 3 pairs are
examples of three broad categories which we’ll cover next:

* Structural types
* Types created using algebraic operators
* Special-cased types (literals, enums and tuples)

(Note that generics are a complex topic that covers both structural
and algebraic types so I’ll not cover them here.)

## Structural Assignability

Structural assignability is an feature of Typescript that most
languages do not have. But it's expensive: it's actually the last
assignability check because it's so slow and painstaking. Simple
equality is the first check! Structural assignability
functions as a fallback when all other kinds of assignability have
failed to return true.

Structural assignability applies to anything that has
properties and signatures: classes, interfaces, and object literal
types, basically. Intersection types also check structural
assignability if no other comparison works.

The comparison itself is not that complicated. It first checks the
target properties that every property in the target is present
in the source:

```ts
var source: { a, b };
var target: { a, b, c};
target = source;
```

This fails because `source` doesn't have a property named `c`. On
the other hand, the source is allowed to have extra properties; the
excess prosperity check happens earlier and only applies to inline
object literals:

```ts
var source: { a, b, c };
var target: { a, b };
target = source;
```

Then matching properties recursively check that their types are
assignable from source to target:

```ts
var source: { a: number, b: string };
var target: { a: number };
target = source;
```

Here, while checking `a`, we make a recursive call to check whether
`number` is assignable to `number`. Of course, this returns true as
soon as it hits the simple equality fast path. But other types may
recur several times before succeeding or failing:

```ts
var source: { a: { b: { c: null } } };
var target: { a: { b: { c: string } } };
target = source;
```

Call, construct and index signatures all work similarly:

```ts
var source: (a: number, b: unknown) => boolean;
var target: (a: never, b: string) => void;
target = source;
```

Except signatures check for correct parameter count instead of missing
properties, and iterate over parameters instead of properties. In
addition, parameters compare contravariantly, which is a fancy way to
say that the direction of source and target flips, so that `string
⟹ unknown` but `(a: unknown) => void ⟹ (a: string) => void`. That's
because when you assign a callback variable, you're not actually
assigning the *parameters* directly. You're assigning a callable thing
to another callable thing. For example:

```ts
var source: string;
var target: unknown;
target = source; // fine, target can be anything, including a string

var source: (a: string) => void;
var target: (a: unknown) => void;
target = source;
target(1); // oops, you can't pass numbers to source
```

### Inheritance

Once you have structural assignability, you don't actually need an
inheritance relation. You just check that derived classes are
assignable to their base classes. Same thing goes for implements. In
other words, this:

```ts
class Base {
  constructor(public b: number) {
  }
}
class Derived extends Base {
  b = 12
  constructor(public d: string) {
    super(this.b);
  }
}
```

Is treated just like this:

```ts
var derived: { b: number, d: string };
var base: { b: number };
base = derived;
```

Of course, the check caches its results so that once you know that a
`Derived` is assignable to a `Base`, you don't have to run the
expensive check again. More on that in the Tricks and Hacks section.

TODO: Maybe point out that private properties are special-cased to
behave nominally?

## Algebraic Assignability

Types with algebraic type operators can be checked algebraically.
Sometimes this check is merely faster than a structural check,
but if you involve type variables, algebraic checks can succeed where
a structural check would fail. For example:

```
function f<T>(source: T | never, target: T) {
   target = source;
}
```

The checker knows that `never` does not change a union it's part of,
so it can eliminate it and recur with a smaller type:

> `T | never ⟹ T`  
> `T ⟹ T`


And since `T===T`, `T | never ⟹ T`.

Other kinds of relations are pretty obvious. For example, when a union
is the target, the source is assignable when it's assignable to any
element of the union:

> `B ⟹ A | B | C`  
> `B ⟹ B`

But when a union is the source, *every* element needs to be assignable
to the target. So `A | B | C ⟹ B` isn't true in general, just in
cases where the target is supertype of all elements of the unions:
`'x' | 'y' | 'z' ⟹ string`.

More complicated relations exist as well. For example, intersection
distributes across union:

> `A & (B | C) ⟹ (A & B) | (A & C)`  
> `(A & B) | (A & C) ⟹ (A & B) | (A & C)`  

Mapped types, index access types, and keyof types also interact in a
number of ways. First, the keyof S is assignable to the
keyof T if T is assignable to S:

> `keyof S ⟹ keyof T`  
> `T ⟹ S`

You can see why the direction flips with an example:

> `S = { a, b, c }`  
> `T = { a, b }`  

Here, the source has an extra property, so it's OK to assign it to a
target with fewer properties. But for the keys, you have to assign the
source's `"a" | "b" | "c"` to the target's `"a" | "b"` because the
smaller union is missing `"c"`.

For mapped types, any type T is assignable to the identity mapping:

> `T ==> { [P in X]: T[P] }`

This follows from the definition of mapped types. Note that the
reverse is not true because mapping a type loses some information.

Finally, some complex relations involve all three kinds of type. For
example, if you have some key type `K` and a rewrite type `X`, a
mapped type `{ [P in K]: X }` is assignable to `T` if `K` is
assignable to the keys of `T` and `X` is assignable to all the
properties of `T`:

> `{ [P in K]: X } ⟹ T`
> `keyof T ⟹ Q` and `X ⟹ T[K]`  

This is true in the other direction too:

> `T ⟹ { [P in K]: X }`  
> `K ⟹ keyof T` and `T[K] ⟹ X`

You can see this is true if you try it with some example types:

> `T = { a: C | D, b: C | D, c: C | D }`  
> `K = "a" | "b"`  
> `X = C | D`  
>  
> `{ a: C | D, b: C | D, c: C | D } ⟹ { [P in "a" | "b"]: C | D }`  
> `{ a: C | D, b: C | D, c: C | D } ⟹ { "a": C | D, "b": C | D }`

## Hacks

In between simple equality and structural comparison, Typescript has
quite a few special cases to handle specific types quickly.

1. Unit and primitive types
2. Excess properties
3. Weak types
4. Maybe cutoff

### Simple Types

Primitive types and unit types are all compared with a long list of
specific rules. The comparisons are quite fast because all of these
types are marked with a bit flag. A few of the rules are odd because
of historical restrictions. For example, any number is assignable to a
numeric enum, but this is not true for string enums. Only strings that
are known to be part of a string enum are assignable to it.

Here are the types that are compared with simple rules:

* string
* number
* boolean
* symbol
* object
* any
* void
* null
* undefined
* never
* string enums
* number enums
* string literals
* number literals
* boolean literals

Note that both the source and target have to be simple for a simple
comparison to succeed. If the source is a string and the target is
some object type, then the compiler will have to use structural
comparison, and it will *first* have to get all the string's methods
and properties.

### Excess property checks

The excess property check and weak type check run right after the
simple type check. The reason they run so early is that, although they
are expensive, if they end up failing an assignability check early,
they save time compared to doing a full structural comparison.

TODO: Discuss how they actually work OK.

TODO: In the hacks section discuss the Maybe cutoff. Also point out
that multiple variants exist: specifically, discuss how type ID works with
generics in the principle non-id variant.
