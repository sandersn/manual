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

This code has three assignability checks:

1. `x = y`, which checks that `y` can be assigned to `x`.
2. `return s.length`, which checks that `s.length` can be returned
from `f`.
3. `f(false)`, which checks that `false` can be assigned to `f`'s first parameter.

To make this happen, we find the types of `x`, `y`, and `s.length` as well as
the parameter and return types of `f`. This allows us to make the
following judgements:

1. `y: number` is assignable to `x: number`.
2. `s.length: number` is assignable to `f`'s return `: number`.
3. `false: boolean` is not assignable to `s: string`.

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

In fact, one common bug in the compiler is to use `===` where
`⟹` is the correct check. This is a bug because Typescript
supports many more types, and kinds of types, than just the primitive
types. Good examples include classes and interfaces, unions and
intersections, and literal types and enums. Those 3 pairs are
examples of three categories which we’ll cover next:

* Structural types
* Types created using algebraic operators
* Special-cased types (literals and enums)

(Note that generics are a complex topic that covers both structural
and algebraic types so I’ll not cover them here.)

## Structural Assignability

Structural assignability is an feature of Typescript that most
languages do not have. But it's expensive: it's the last assignability
check because it's so slow and painstaking. Structural assignability
functions as a fallback when all other kinds of assignability have
failed to return true. In fact, `===` is the first check!

Structural assignability applies to anything that has
properties and signatures: classes, interfaces, and object literal
types, basically. Intersection types also check structural
assignability if no other comparison works.

The comparison itself is not that complicated. It first checks that
every property in the target is present in the source:

```ts
var source: { a, b };
var target: { a, b, c};
target = source;
```

But `{ a, b } ⟹ { a, b, c }` is not true because `source` doesn't have
a property named `c`. On the other hand, the source is allowed to have
extra properties, so `{ a, b, c } ⟹ { a, b }` is true:

```ts
var source: { a, b, c };
var target: { a, b };
target = source;
```

If every source property has a matching target property, then matching
properties recursively check that their types are assignable from
source to target:

```ts
var source: { a: number, b: string };
var target: { a: number };
target = source;
```


Here, while checking `a`, we make a recursive call to check whether
`number` is assignable to `number`. We can write this like so:

> `{ a: number, b: string } ⟹ { a: number }`  
> `number ⟹ number`

Of course, the second check returns true as soon as it hits the `===`
fast path. But other types may recur several times before succeeding
or failing:

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
target = source; // should be an error, because:
target(1); // oops, you can't pass numbers to source
```

### Inheritance

Once you have structural assignability, you don't actually need an
inheritance relation. You just check that derived classes are
assignable to their base classes. Same thing goes for implements. In
other words, if you have classes like this:

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

Variable declarations that use these types:

```ts
var derived: Derived;
var base: Base;
base = derived;
```

Are compared just like this:

```ts
var derived: { b: number, d: string };
var base: { b: number };
base = derived;
```

There is no inherent relation between `Derived` and `Base`. Of course,
the assignability check caches its results so that once you know that
a `Derived` is assignable to a `Base`, you don't have to run the
expensive check again. In fact, this check runs as soon as the
compiler sees `class Derived extends Base`, so the structural approach
ends up costing about the same as the nominal one.

## Algebraic Assignability

Types with algebraic type operators can be checked algebraically.
Sometimes this check is merely faster than a structural check, but if
the type contains type variables, algebraic checks can succeed where a
structural check would fail. For example:

```
function f<T>(source: T | never, target: T) {
   target = source;
}
```

The checker knows that `never` does not change a union it's part of,
so it can eliminate it and recur with a smaller type:

> `T | never ⟹ T`  
> `T ⟹ T`


And since `T===T`, `T | never ⟹ T` is true.

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
> `keyof T ⟹ K` and `X ⟹ T[K]`  

This is true in the other direction too:

> `T ⟹ { [P in K]: X }`  
> `K ⟹ keyof T` and `T[K] ⟹ X`

You can see this is true if you try it with some example types:

> `T = { a: C, b: D, c: C | D }`  
> `K = "a" | "b"`  
> `X = C | D`  
>  
> `T ⟹ { [P in K]: X }`  
> `{ a: C, b: D, c: C | D } ⟹ { [P in "a" | "b"]: C | D }`  
> `{ a: C, b: D, c: C | D } ⟹ { "a": C | D, "b": C | D }`

is true whenever the following two things are true:

> `K ⟹ keyof T`  
> `"a" | "b" ⟹ "a" | "b" | "c"`  

and

> `T[K] ⟹ X`  
> `T["a" | "b"] ⟹ C | D`  
> `C | D ⟹ C | D`

## Hacks and Tricks

In between simple equality and structural comparison, Typescript has
quite a few special cases to handle specific types quickly.

1. Unit and primitive types
2. Excess properties and weak types
3. Recursion cutoffs

### Simple Types

Primitive types and unit types are all compared with a long list of
specific rules. The comparisons are quite fast because all of these
types are marked with a bit flag, so the typical line of code looks
like:

```ts
if (source.flags & TypeFlags.StringLike && target.flags & TypeFlags.String) return true;
```

A few of the rules are odd because of historical restrictions. For
example, any number is assignable to a numeric enum, but this is not
true for string enums. Only strings that are known to be part of a
string enum are assignable to it. That's because numeric enums existed
before union types and literal types, so their rules were originally
looser.

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

### Excess property and weak type checks

The excess property check and weak type check run right after the
simple type check. The reason they run so early is that, although they
are expensive, if they end up failing an assignability check early,
they save time compared to doing a full structural comparison.

The excess property check applies only to the types of object
literals. Its only purpose is to fail otherwise-legal assignments.
With structural assignability, it's fine to have extra properties, so
`{ a, b } ⟹ { a }`. However, when you assign an object literal to a
variable with a type like `{ a }`, it's very unlikely that you want to
include properties besides `a`. So the excess property check disallows
this assignment.

The weak type check applies only if the source contains nothing but
optional properties. With structural assignability, *any* type is
assignable to a weak type. All these assignments are structurally
sound:

> `{ a } ⟹ { a?, b? }`
> `{ a, b, e } ⟹ { a?, b? }`
> `{ c, d } ⟹ { a?, b? }`
> `{ } ⟹ { a?, b? }`

However, only the first two make any kind of sense, so the weak type
check requires that the source share at least one property with a weak
target.

### Structural cutoffs

Structural assignability has problems with recursion. Specifically, if
you write a class like this:

```ts
declare class Box<T> {
  t: T
  m(b: Box<T>): void
}
var source = new Box<string>();
var target = new Box<unknown>();
target = source;
```

To find out whether target is assignable to source:

> `Box<string> ⟹ Box<unknown>`  
> `{ t: string, m(b: Box<string>): void } ⟹ { t: unknown, m(b: Box<unknown>): void }`  

Now we have to show that the types of `t` and `m` are assignable.
`t` is easy enough:

> `string ⟹ unknown`

But `m` poses a problem:

> `(b: Box<string>) => void ⟹ (b: Box<unknown>) => void`  
> `Box<string> ⟹ Box<unknown>`  
> `{ t: string, m(b: Box<string>): void } ⟹ { t: unknown, m(b: Box<unknown>): void }`  

Oh no. Looks like we are going to loop forever!

Typescript actually has two solutions for this problem. The simplest
is to notice that `Box === Box` and to check the arguments of `Box`
instead of using structural assignability:

> `Box<string> ⟹ Box<unknown>`  
> `string ⟹ unknown`  

This works quite often because most of the time people write nominal
code, even in a structural type system. But what if somebody shows up
with a similar class that they then updated to work with `Box`?

```ts
declare class Ref<T> {
  private item: T
  deref(): T
  // for Box compatibility:
  readonly t: T
  m(b: Ref<T>): void
}
```

Now, if the source is `Ref<string>` and target is `Box<unknown>`, we
have to compare the whole thing structurally. Everything goes well
until we try to check `other` again:

> `Ref<string> ⟹ Box<unknown>`  
> ...  
> `(b: Ref<string>) => void ⟹ (b: Box<unknown>) => void`  
> `Ref<string> ⟹ Box<unknown>`  

Even though `Ref !== Box`, we can use the fact that this is the second
occurrence of `Ref<string> ⟹ Box<unknown>` to infer that the second
check will not provide any new information. However, instead of
returning true, structural assignability actually uses a ternary to
return Maybe. A Maybe result becomes true if all non-Maybe results are
true. Otherwise it stays Maybe.

This causes `Ref<string> ⟹ Box<unknown>` to succeed because `Ref.t` is
in fact assignable to `Box.t` and `item` and `deref` don't matter for
assignability.

However, you can *still* write types that these two checks don't
catch. Specifically:

```ts
declare class Functor<T> {
  fmap<U>(f: (t: T) => U): Functor<U>;
}
declare class Mappable<T> {
  fmap<U>(f: (t: T) => U): Mappable<U>;
}
```

In this case, for `Functor<string> ⟹ Mappable<unknown>`, you end up
having to check `Functor<U> ⟹ Mappable<U>` while checking the return
type of `fmap`. Once you get started checking
`Functor<U> ⟹ Mappable<U>`, you're stuck: every recursive check of
`fmap` creates a fresh `U`, so you have to check
`Functor<U> ⟹ Mappable<U>` for this fresh `U`.

To prevent this, structural assignability has an arbitrary 5-deep
cutoff for comparing identical pairs of types, even if their arguments
are different. That is, if we compare the pair (Functor, Mappable) 5
times, that particular comparison will return Maybe.

## Summary

In order to present concepts in order of least to most complex, I
presented the parts of the assignability check in a different order
than they actually occur. The actual algorithm proceeds as follows:

1. If source === target, return true.
2. If source and target are simple and are assignable, return true.
3. If source is an object type and the target has excess properties, return false.
4. If source type is weak and the target shares no properties, return false.
5. If the source or target types are algebraic, simplify the types if possible and recur.
6. If the source is structurally assignable to the target, return true.
7. Otherwise, return false.

As we saw in the previous section, step 6 might return "Maybe", which
counts as true.

I also skipped quite a few small details like how private properties
are handled. For the actual code, look at `checkTypeRelatedTo` in
src/compiler/checker.ts in the TypeScript repository on github.
