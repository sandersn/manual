# How to Upgrade to TypeScript Without Anybody Noticing

OK, well, actually this means you have to stay with Javascript, but
check with Typescript. And you'll find it really hard to stay truly
incognito, because you'll want to add devDependencies for various
typings.


## Add tsconfig

I start with `tsc --init` and set some settings.

I added resolveJsonModule since the code requires a json file at one point.

I ended up with this:

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "commonjs",
    "allowJs": true,
    "checkJs": true,
    "noEmit": true,
    "resolveJsonModule": true
    "esModuleInterop": true
    "strict": false,
  },
  "exclude": [
    "tests/fixtures",
    "tests/integration"
  ]
}
```

For Javascript code, target, module, esModuleInterop and noEmit should
pretty much always have these values. And if your code ever `require`s
a JSON file, then you'll want resolveJsonModule too, so that
Typescript will analyze it too.

*Officially*, strict should be false, but as you will see later,
 strict mode can be pretty great for Javascript code, not just
 Typescript code.

For typescript-eslint-parser, I added an exclude list, since I wanted
to check all the source files. But it turns out the parser's tests
take the form of JS files themselves, and I don't want to check those.
Most of them are malformed in some way.

You might want to use "include" if, say, you only want to check your
source and not your tests or scripts. Or you can use "files" to give
an explicit list of files to use, but this is annoying except for
small projects.

Run tsc and make sure it prints out errors. Now open up
files in your editor and make sure the same errors show up there.

Congratulations! You've done the only *required* part. Everything else
just helps reduce the number of errors you see when editing.

## Add @types packages.

Your first order of business is to install types for packages you use.
This is the easiest way to reduce the number of errors. Basically, if
you use package X, and you see errors when you use it, try installing
`@types/X`. Tons of packages have definitions, so you will probably
find most of your dependencies are typed.

Note that if you use node at all, you likely need `@types/node`.

For typescript-eslint-parser, I installed:

* @types/node
* @types/jest
* @types/estree
* @types/shelljs
* @types/eslint-scope

If you are really, truly, completely in stealth mode, make sure not to
save these installations in package.json. That's a bit extreme,
though, isn't it?

Anyway, these two types packages still didn't work:

* @types/shelljs
* @types/estree

They fail for different reasons, though. shelljs's types are just
missing types for 'shelljs/make'. estree *has* all the correct types,
but the types aren't imported correctly. So now you can decide where
to start fixing errors.

You can

1. Add shelljs' missing types. TODO: LINK
2. Fix up the type annotations that refer to estree types. TODO: LINK
3. Add missing typings for your dependencies. TODO: LINK
4. Add missing typings in your code. TODO: LINK
5. Fix errors in existing types. TODO: LINK

6. Stub out modules in a d.ts. (When you don't mind checking in
Typescript code, although others don't necessarily need to edit it.)

## Missing types in dependencies

In Makefile.js, I see a few errors. The first is that the module
`require('shelljs/make')` isn't found. The second group of errors is
that the names 'find', 'echo' and a few others aren't found.

These errors are related. It turns out that shelljs doesn't even
include shelljs/make right now. It's completely missing. I looked it
up and shelljs/make does two things:

1. Add a global object named 'target' that allows you to add make
targets.
2. Add the contents of the parent shelljs to the global scope.

So I want to add this to Definitely Typed. My first step is to create
make.d.ts inside node_modules/@types/shelljs/. This is just for
development, because it makes it super easy to test that I'm actually
fixing the missing stuff.

I start with this:

```ts
import * as shelljs from './';
declare global {
  const cd: typeof shelljs.cd;
  const pwd: typeof shelljs.pwd;
  // ... all the rest ...
}
```

Then I add the type for 'target' to the globals as well:

```ts
const target: {
  all?: Target;
  [s: string]: Target;
}
interface Target {
  (...args: any[]): void;
  result?: any;
  done?: boolean;
}
```

This exposes a couple more errors, which we will look at later.

Now we want to publish this to Definitely Typed.

1. `git clone https://github.com/DefinitelyTyped/DefinitelyTyped`
2. `cp node_modules/@types/shelljs/make.d.ts ~/DefinitelyTyped/types/shelljs/`
3. `git checkout -b add-shelljs-make`
4. Create a PR for the change.

If there are lint problems, the CI run on Travis will catch them.

For more detail on writing definitions for Definitely Typed, [see the
long explanation](?????).

## Fix up existing type annotations

In analyze-scope.js, I see quite a few errors about missing estree
types like Identifier and ClassDeclaration. They're all types
that *do* exist in estree. The problem is that they're not imported,
and there is no Javascript value to import. It's just a type.

I added typedef with import types to import the types:

```js
/** @typedef {import("estree").Identifier} Identifier */
```

You could skip the typedef and write the import type each time, but it
is a lot of typing.

## Define new @types packages

Sometimes the number of missing types is bigger, though. The first
error in analyze-scope.js is on

```js
new PatternVisitor(this.options, node, callback)
```

It says that it expects 0 arguments but got 3. Why did it expect 0?
Well, PatternVisitor doesn't have its own constructor, it extends
the class from 'eslint-scope/lib/pattern-visitor'. If you look at
[the source](TODO LINK), you'll see a class with a 3-parameter
constructor. But this class extends esrecurse.Visitor, which has a
2-parameter constructor. Neither one of these classes has types.
Adding types for nearly all of eslint-scope and esrecurse is a big
task. There must be some way to short-circuit this.

## Define new types

Fixing these types still leaves a lot of undefined types that are
similar to estree types but actually exist only in this project.
For example, TSTypeAnnotation is a type that exists only in
typescript-eslint-parser, not estree. To start with, I defined a lot
more typedefs with any. This got rid of a lot of errors:

```js
/** @typedef {any} TSTypeQuery */
```

Then I changed the typedefs one by one to 'unknown', and went looking
for errors that popped up:

```js
/** @typedef {unknown} TSTypeQuery */
```

Turns out that only one usage exists, with a left property, so I added
that:

```js
/** @typedef {{ left: unknown }} TSTypeQuery */
```

With little knowledge of typescript-eslint-parser, that's as far as I
can go with this, but of course if you work on a project, you'll have
a better idea of what the types should be.

(and use them)
(and decide where to store them)
(CHECK YOUR WORK)

## Fix errors in existing types

After I got the shelljs types working, only two errors were left. It
turns out that the `Function` type isn't specific enough to work with
shelljs' `filter` function. So I changed this helper function:


```js
/**
 * Generates a function that matches files with a particular extension.
 * @param {string} extension The file extension (i.e. "js")
 * @returns {Function} The function to pass into a filter method.
 * @private
 */
function fileType(extension) {
    return function(filename) {
        return filename.substring(filename.lastIndexOf(".") + 1) === extension;
    };
}
```

I changed `@returns {Function}` to the more precise
`@returns {(s: string) => boolean`.



typescript-eslint-parser also needs to define its own TS-specific
types. I started with any:

```js
/** @typedef {any} TSEnumDeclaration */
```

Later I'll change this to unknown to surface errors that will help me
refine the type.

4. Fix errors

This *usually* means adding types. Whenever you find a real bug, it
means fixing that bug.

5. what types mean
6. some normal types you might write
7. some advanced types
8. some next steps if you get buy-in
9. where to learn more I guess

== Some buy-in ==
1. Add tsconfig and check it in
2. Add d.ts to hold non-installed types
3. Add a build task and maybe even hook it up to CI.
