# ThisType

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
// LittleVue is like Vue in that it takes a data section and a methods section
// and makes them available on the top-level and as $data
type LittleVue<D,M> = D & M & {
  $data: D,
  $parent: LittleVue<any, any>
  // other stuff here
}
// LittleVueOptions has a data section and a methods sections,
// but wants to make sure that its 'this' is the type after Vue transforms it.
type LittleVueOptions<D,M> = ThisType<LittleVue<D,M>> & {
  data?: D,
  methods?: M,
}
declare function create<D,M>(options: LittleVueOptions<D,M>): LittleVue<D,M>;

// note that using create requires no type annotations
var app = create({
  data: {
    x: 12
  },
  methods: {
    m1() { // this has the right type here!
      console.log(this.x);
    }
    m2() {
      return this.m1() // this contains 'x', 'm1' and 'm2'
    }
  }
}
app.x // app also contains 'x', 'm1' and 'm2'
app.$data.x // and app.$data contains just 'x'
```

## Why

Why do we use `ThisType<T>` to solve this problem?

We want to specify the type used for inference separately from the
type used for context when the contextual type needs to rely on
inferences made during type inference. But contextual typing
interferes with type inference, sometimes causing it short-circuit to
the wrong answer.

Where there is no dependency, which is the normal case, it doesn't
matter if the two types are the same, and cases where contextual
typing interferes with type inference are hard to notice.

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
