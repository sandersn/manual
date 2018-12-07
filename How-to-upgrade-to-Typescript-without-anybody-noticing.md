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

1. Fix up the type annotations that refer to estree types. TODO: LINK
2. Add shelljs' missing types. TODO: LINK
2. Add missing typings for your dependencies. TODO: LINK
3. Add missing typings in your code. TODO: LINK
3. Fix errors in existing types. TODO: LINK

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


Briefly, I noticed that shelljs has a couple of helper modules named
`make` and `global` that dump all of shelljs' contents into the global
scope. So I added a declaration file that reflects that into the
shelljs typings.


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

Next I looked at analyze-scope.js. The most common errors were related
to types from @types/estree, which weren't imported. You can't
directly import types in Javascript, so I used import
syntax:

```js
/** @typedef {import("estree").Identifier} Identifier */
```

Note that values that have types, like classes and constructor functions.

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
