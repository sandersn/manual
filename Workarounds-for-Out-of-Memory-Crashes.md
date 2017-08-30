# Workarounds for Out of Memory Crashes

First of all, congratulations! You found a bug in the Typescript
compiler! Please [file a bug](https://github.com/Microsoft/TypeScript/issues).
While you're waiting for a fix, here are some workarounds, plus some
background on what is probably going wrong.

The root cause of an out-of-memory bug is usually one of two things:
relating complex recursive types, or analysing control flow. Either of
these can encounter infinite or exponential recursions. With
control-flow analysis, it's usually a simple bug; with type relations,
the behaviour is usually a structural limitation of the compiler.

Note: I update this document periodically to reflect current bugs and
workarounds. Right now the solutions reflect Typescript 2.4 and 2.5.
Previous versions targetted Typescript 2.2 and 2.3; you can see
earlier versions by checking out https://github.com/sandersn/manual.

## Complex Recursive Types

A good example of a complex recursive type is this example based on tsoption:

```ts
interface Some<T> {
  ap<U, V extends Some<(value: U) => U>>(option: V): Some<U>
  chain<U>(f: (value: T) => Some<U>): Some<U>
  chain<U>(f: (value: T) => None<U>): None<T>
}
interface None<T> {
  ap<U, V extends Some<(value: U) => U>>(option: V): None<U>
  chain<U>(f: (value: T) => Some<U>): None<T>
  chain<U>(f: (value: T) => None<U>): None<T>
}

type Option<T> = Some<T> | None<T>
```

Notice how `Some` and `None` return instances of each other from each
of their methods. The instances are even unique per method, because
each method has one or two type parameters.

### noStrictGenericChecks

The first workaround you should try is `"noStrictGenericChecks": true`.
Does compilation now succeed?
This disables additional checking that was added in Typescript
2.4. With noStrictGenericChecks: true, compile time and memory usage
should return to their 2.3 levels. But you miss out on the additional
checking, so this is still just a workaround.

### Avoid type aliases

If you want to use the stricter generic checking in 2.4, avoid type
aliases in favour of interfaces. When checking type relationships,
caching and early exit is more effective for interfaces. For example:

```ts
// don't write this:
type Some<T> = MonadSome<T> & {
  flatMap ...
}

// instead, write this:
interface Some<T> extends MonadSome<T> {
  flatMap ...
}
```

This example is once again based on tsoption.

## Large Javascript

When you use `"allowJs": true`, Typescript may try to compile Javascript
packages inside `node_modules`. Large Javascript libraries still
crash Typescript sometimes, often because of expanding types (see next
section). In general, Typescript works harder on unannotated code to
infer types, which of course means that large Javascript libraries
stress it the most.

### Debugging

Do you have `"allowJs": true` in your tsconfig? Turn it off. Does
compilation complete, even if it gives tons of errors? If there is no
crash, then the culprit is the compiler getting stuck on Javascript code.

### Workaround

1. Install typings for libraries you use. The compiler will stop
   looking as soon as it sees the `.d.ts` and won't even try to process
   `.js` or `.ts` files.

    This is the best solution. You should do this for your
    dependencies anyway to get better type information. The simplest
    thing to try is just `npm install --save-dev @types/packageName`
    after you run `npm install --save packageName`.

1. Specifically `"exclude"` node_modules or `"include"` your source
directory.

    This will help the compiler to know that it doesn't need to
    process anything but your source code. For example, you might have
    this tsconfig:

    ```ts
    {
      "compilerOptions": {
        "allowJs": true
      },
      "include": [ "src" ]
    }
    ```

    Or this one:

    ```ts
    {
      "compilerOptions": {
        "allowJs": true
      },
      "exclude": [ "node_modules" ]
    }
    ```

## Expanding types

Typescript can infer the types of some declarations based on the way
they are used:

```ts
let x
let y = null
let l = []
x = 1 // x is now number
x = 'foo' // x is now string
y = 1 // y is now number
l.push(1) // l is now number[]
l.push('foo') // l is now (string | number)[]
```

This analysis is only turned on when `"noImplicitAny": true`. It
requires the compiler to look at usages of a variable as well as its
declaration to figure out its type. This necessarily takes more time
and sometimes takes a *lot* more time when it hits a bug.

### Debugging

Do you have `"noImplicitAny": true`? Turn it off. If compilation
completes, then control flow analysis is the culprit.

### Workaround

Look for un-annotated variables that are not initialised (`let x`) or
initialised to `null`, `undefined` or `[]`. Add annotations or give a
more useful initial value to them.

```ts
let l = []
// instead try
let l: number[] = [];
// -or-
let l = [1, 2, 3]
```

### Webpack

Webpack and ts-loader have an option `"transpileOnly": true` that
tries makes the compiler skip as much checking as possible.
Unfortunately, some control flow analysis still happens, so it's
possible to hit an out-of-memory crash in this mode. The workaround is
to turn off `transpileOnly` or to give variables an initial value
that's not `null/undefined/[]`.
