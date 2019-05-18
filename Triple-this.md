# Triple `this`

Javascript's treatment of `this` is confusing and, on the surface, inconsistent. Newer version of the language add arrow functions to avoid the worst problem with `this`, but that doesn't make the language any *simpler*.

Typescript has three features with `this`. These features don't make the language simpler either, but at least your code will do the right thing by default: 

* `this` parameters, which let you specify the type of `this` in an old-style constructor function.
* `this` types, which let you easily writing chaining methods, also called a *fluent API*.
* `ThisType<T>`, which is intended for complex factory functions that take an object literal. It lets the factory manipulate `this` of the object literal.

## `this` Parameters

Helps you specify the type of this.

You only need this if you have `this` outside classes.

You can turn on `--noImplicitThis` to get errors wherever you need to add them.

## `this` Types

`this` types are fairly simple to use. If you want to chain methods, and also have subclasses, you can return `this` from your methods:

```ts
declare class Collection<T> {
    filter(f: (t: T) => boolean): this;
}
declare class List<T> extends Collection<T> {
    constructor(ts: T[]);
    nth(n: number): T;
}
const x = new List<string>(['a', 'b', 'c']).filter(s => s.length > 0).nth(2);
```

Notice that `filter` is `Collection.filter`, but `nth` is `List.filter`. If `filter` returned `Collection<T>`, `nth` would not be found on the return type of `filter`.

Overuse of `this` can be expensive.
They are equivalent to a hidden type parameter that's passed on construction:

```ts
declare class B {
    m(): this;
}
declare class C extends B {
}
let c = new C();
c = c.m();
```

is equivalent to this code:

```ts
declare class B<This> {
    m(): This;
}
declare class C<This> extends B<This> {
}
let c = new C<C>();
c = c.m();
```

Class hierarchies that are deep, with big base classes, cause lots of instantiations of lots of classes to occur. This is expensive and can lead to long compile times.

They don't work well with intersections since each intersection retains its own `this` type.

## ThisType&lt;T&gt;

Typescript 2.3 adds a special marker type called `ThisType<T>`, which
lets you control the contextual type of `this`.

I'll explain **what** it is, **why** it exists and **how** it works.

## What

Intersect `ThisType<T>` with another type, and `T` will be used for contextual
typing of `this`. This will apply when contextually typing something,
like an object literal or a function literal, that makes `this`
available. People rarely specify the type of `this`, so it's useful for
the type to come from context.

For example:

```ts
// View is like Vue in that it takes a data section and a methods section
// and makes them available on the top-level and as $data
type View<D, M> = D & M & {
  $data: D,
  $parent: View<any, any>
  // other stuff here
}
// LittleVueOptions has a data section and a methods sections,
// but wants to make sure that its 'this' is the type after Vue transforms it.
type ViewOptions<D, M> = ThisType<ViewOptions<D, M>> & {
  data?: D,
  methods?: M,
}
declare function create<D, M>(options: ViewOptions<D, M>): View<D, M>;

// note that using create requires no type annotations
var app = create({
  data: {
    x: 12
  },
  methods: {
    m1() { // 'this' has the right type here!
      console.log(this.x);
    }
    m2() {
      return this.m1() // 'this' contains 'x', 'm1' and 'm2'
    }
  }
}
app.x // 'app' also contains 'x', 'm1' and 'm2'
app.$data.x // and 'app.$data' contains just 'x'
```

## Why

Why do we use `ThisType<T>` to solve this problem?

In order to avoid requiring type annotations, we want to use
contextual typing to provide a `this` type. Like this:

```ts
declare function registerCallback(cb: (this: Element) => boolean): void;
registerCallback(function () {
  // the callback contextually types 'this' as 'Element'
  console.log(this.attributes);
  return true;
};
```

However, this doesn't work when we are trying to infer a type variable
at the same time for the type at the same time:

```ts
type MapOfThisMethods<This> = { [s: string]: (this: This, ...args: any[]) => any };
declare function create<M extends MapOfThisMethods<M>>(options: ViewOptions<M>): View<M>;
```

Notice that `M` is used in its own constraint! This is illegal. You
can try to push this around but it turns out you can't get rid of it entirely.

## How

When looking up the type of `this` in a function, the checker looks
for a contextual type if there is no type annotation. Finding the
contextual `this` type requires finding the contextual type of the
function. If the contextual type of the function includes
`ThisType<T>`, or some object-literal parent of the function includes
`ThisType<T>`, then the contextual type of `this` is `T`. See
`getContextualThisParameterType` in the checker, which is called by `checkThisExpression`.

## Any questions?

I am @sandersn on Github and @sanders_n on Twitter.
