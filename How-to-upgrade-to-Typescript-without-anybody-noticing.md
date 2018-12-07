# How to Upgrade to TypeScript Without Anybody Noticing

OK, well, actually this means you have to stay with Javascript, but
check with Typescript. And you'll find it really hard to stay truly
incognito, because you'll want to add devDependencies for various
typings.


1. Add tsconfig (and add to .gitignore)

I start with `tsc --init` and then turn on allowJs and checkJs and noEmit, and
turn off strict, and set target to esnext.

I added resolveJsonModule since the code requires a json file at one point.

Add files to ignore if needed; I had to ignore 'tests/fixtures' since
this is a parser project with lots of invalid code in files.

OR Add files to include if needed. I added 'lib' and 'tests' since
those made up the source code.

Run tsc and make sure it prints out errors or something. Also open up
files in your editor and make sure the same errors show up there. (You
should have errors. If you don't something is either really wrong or
really right.)

Congratulations! You've done the only *required* part. Everything else
just helps reduce the number of errors you see when editing.

2. Update package.json with @types.

You'll have lots of errors. Because most packages have types on
Definitely Typed, the first thing to do is to install types.

Start installing missing packages. Try `npm install
@types/packagename` first.

Note that if you target node at all, you likely need `@types/node`.

I had to install

* @types/node
* @types/jest
* @types/estree

This still didn't fix

* @types/shelljs, which doesn't export any globals
* @types/estree, since its types aren't imported anywhere

OK, with this second step done, you have some choices about how to
start fixing errors.

You can

1. Fix up incorrect annotations. TODO: LINK
2. Add missing typings for your dependencies. TODO: LINK
3. Add missing typings in your code. TODO: LINK

3. Look for and fix errors

Run tsc again.
Open a file with some errors.

I tried Makefile.js, since it was first in the list. Unfortunately,
installing shelljs didn't work because the part of shelljs that the
code uses isn't documented by the typings.


I fixed this by submitting a PR to Definitely
Typed. The documentation on how to do that is [here](????????).
Briefly, I noticed that shelljs has a couple of helper modules named
`make` and `global` that dump all of shelljs' contents into the global
scope. So I added a declaration file that reflects that into the
shelljs typings. It looked like this:

```ts
import * as shelljs from './';
declare global {
  const cp: typeof shelljs.cp;
  const pushd: typeof shelljs.pushd;
  // etc ...
}
```

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
