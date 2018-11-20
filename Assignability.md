## Table of Contents

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
`number`. The third check is that the type of false, `boolean` is assignable to
`string`. Which it isn’t.

For a simple type system consisting only of the primitive types,
assignability would just be equality:

```ts
function isAssignableTo(source: Type, target: Type): boolean {
  return source === target;
}
```

For brevity's sake, I'll often use the operator &xrArr; to represent
`isAssignableTo`, like this:

number &xrArr; number.

In fact, one common bug in the compiler is to use equality where
assignability is the correct check. This is a bug because Typescript
supports many more types, and kinds of types, than just the primitive
types. Good examples include classes and interfaces, unions and
intersections, and literal types and tuples. Those 3 pairs are
examples of three broad categories which we’ll cover next:

* Structural types
* Types created using algebraic operators
* Special-cases types (literals, enums and tuples)

(Note that generics are a complex topic that covers both structural
and algebraic types so I’ll not cover them here.)

# Structural Assignability

Structural assignability is an feature of Typescript that most
languages do not have. But it's expensive: it's actually the last
assignability check because it's so slow and painstaking. Simple
equality is actually the first check! Structural assignability
functions as a fallback when all other kinds of assignability have
failed to return true.

Structural assignability applies to anything that actually
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
say that the direction of source and target flips, so that string
&xrArr; unknown but (a: unknown) => void &xrArr; (a: string) => void. That's
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

TODO: Discuss how we don't need subtype relations (including
implements) because of structural comparison.
TODO: Probably don't discuss overloads.

TODO: In the hacks section discuss the Maybe cutoff. Also point out
that multiple variants exist. And discuss how type ID works with
generics.
