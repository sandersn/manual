# How to Upgrade to TypeScript Without Anybody Noticing

This guide will show you how to upgrade to Typescript without anybody
noticing. Well, people *might* notice &mdash; what I really mean is
that you won't have to change your build at all. You'll have the
ability
[to get errors and completions in supported editors](https://github.com/Microsoft/TypeScript/wiki/TypeScript-Editor-Support)
and to get errors on the command line from `tsc`, the Typescript
compiler, but you won't have to integrate Typescript into your build.

Here's what you'll actually need to check in to source control:

1. A Typescript configuration file, `tsconfig.json`.
2. New dev dependencies from the `@types` package.
3. A Typescript declaration file to hold miscellaneous types.

How can you upgrade with so little change? Well, the secret is that
*you're not using Typescript*. The Typescript compiler can check
Javascript just fine, so you can stick with Javascript and use JSDoc to
provide type information. This is less convenient than Typescript's
syntax for types, but it means that your files stay plain old
Javascript and your build (if any) doesn't change at all.

Let's use
[typescript-eslint-parser](https://github.com/eslint/typescript-eslint-parser)
as an example package so you can follow along if you want.
Confusingly, even though the name includes "Typescript", the parser is
actually written in Javascript, although it has since been merged into
[a larger project that is written in Typescript](https://github.com/typescript-eslint/typescript-eslint).

This guide assumes that you have used Typescript enough to:

1. [Have an editor set up to work with Typescript](https://github.com/Microsoft/TypeScript/wiki/TypeScript-Editor-Support).
2. Have used npm to install a package.
3. [Know the basic syntax of type annotations](http://2ality.com/2018/04/type-notation-typescript.html) &mdash; `number`,
`{ x: any }`, etc.

If you want to look at the package after the upgrade, you can run the
following commands, or
[take a look at the branch on github](https://github.com/eslint/typescript-eslint-parser/compare/master...sandersn:add-tsconfig):

```sh
git clone https://github.com/sandersn/typescript-eslint-parser
cd typescript-eslint-parser
git checkout add-tsconfig
npm install
```

Also make sure that you have typescript installed on the command line:

```
npm install -g typescript
```

## Add tsconfig

Your first step is to start with `tsc --init` and change the settings
in the `tsconfig.json` that it produces. There are other ways to get
started, but this gives you the most control. Run this command from
the root of the project:

```sh
tsc --init
```

Here is what you should end up with, skipping the lines you don't need
to change:

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "noEmit": true,
    "target": "esnext",
    "module": "commonjs",
    "resolveJsonModule": true,
    "strict": false
  },
  "exclude": [
    "tests/fixtures/",
    "tests/integration/"
  ]
}
```

For Javascript, your "compilerOptions" will pretty much always look
like this.

* `allowJs` &mdash; Compile JS files.
* `checkJs` &mdash; Give errors on JS files.
* `noEmit` &mdash; Don't emit downlevel code; just give errors.
* `target` &mdash; Target the newest version of EcmaScript since we're
  not emitting code anyway.
* `module` &mdash; Target node's module system since we're not emitting
  code anyway.
* `resolveJsonModule` &mdash; Compile JSON files (if they're small enough).
* `strict` &mdash; Don't give the strictest possible errors.

A few notes:
* `"target"` and `"module"` should not actually matter since they have to do
with generated downlevel code, but you'll get some bogus errors if you
use ES6 classes like `Map` or `Set`.
* `"resolveJsonModule"` is optional, but you'll need it if your
code ever `require`s a JSON file, so that Typescript will analyze it
too.
* `"strict"` should be false by default. You can satisfy the compiler on
  strict mode with pure Javascript, but it can require some odd code.

You may want to specify which files to compile. For
typescript-eslint-parser, you'll be happiest with an `"exclude"` list.
You *may* want to check *all* the source files, which is what you get
by default. But it turns out that checking a Javascript parser's tests
is a bad idea, because the tests are themselves malformed Javascript
files. Those malformed test files shouldn't be checked and mostly
don't parse anyway.

You might want to use `"include"` if, say, you only want to check your
source and not your tests or scripts. Or you can use `"files"` to give
an explicit list of files to use, but this is annoying except for
small projects.

OK, you're all set. Run `tsc` and make sure it prints out errors. Now open up
files in your editor and make sure the same errors show up there.
Below are the first few errors you should see:

```
Makefile.js(55,18): error TS2304: Cannot find name 'find'.
Makefile.js(56,19): error TS2304: Cannot find name 'find'.
Makefile.js(70,5): error TS2304: Cannot find name 'echo'.
```

You should be able to see the same errors when you open Makefile.js in
your editor and look at lines 55 and 56.

Congratulations! You've done the only *required* part of the upgrade.
You can check in `tsconfig.json` and start getting benefits from
Typescript's checking in the editor without changing anything else. Of
course, there are a huge number of errors, hardly any of which are due
to real bugs. So the next step is to start getting rid of incorrect
errors and improving Typescript's knowledge of the code.

[Here's the commit.](https://github.com/eslint/typescript-eslint-parser/commit/9ee85f151b0ef81fa592ddbdb4f60aeb842ae42c)

## Install @types packages.

Your first order of business is to install types for packages you use.
This allows Typescript to understand them, which makes it the easiest
way to reduce the number of errors. Basically, if you have a
dependency on some package, say, `jquery`, and you see errors when you
use it, you probably need a dev dependency on `@types/jquery`. The
type definitions in `@types/jquery` give Typescript a model of jquery
that it can use to provide editor support, even though jquery was
written before Typescript existed.

[Definitely Typed](https://github.com/DefinitelyTyped/DefinitelyTyped)
is the source for the packages in the `@types` namespace. Anybody can
contribute new type definitions, but tons of packages already have
type definitions, so you will probably find that most of your dependencies
do too.

[Here's a good starting set for typescript-eslint-parser](https://github.com/eslint/typescript-eslint-parser/commit/0a8bf69fc1d8c0967e7e67ade2fec38ddfeefeda),
although there are likely more available:

```sh
npm install --save-dev @types/node
npm install --save-dev @types/jest
npm install --save-dev @types/estree
npm install --save-dev @types/shelljs@0.8.0
npm install --save-dev @types/eslint-scope
```

After the installation, these three types packages still didn't work
(although notice that we intentionally installed an old version of
shelljs -- more on that later):

* `@types/shelljs`
* `@types/eslint-scope`
* `@types/estree`

They fail for different reasons, though. shelljs and es-lint-scope just
don't have types for a lot of their values. estree *has* all the
correct types, but the types aren't imported correctly.
[Part 2](How-to-upgrade-to-Typescript-without-anybody-noticing-part-2.md)
shows how to fix these two problems.

At this point, you have types for some of your packages working and
your own code checked by the Typescript compiler. The next step is to
fix compile errors in the rest of the package types, or to fix them in your
own code. Or you can just ignore the errors and start using the
Typescript support in the editor.

Next up:
[Part 2](How-to-upgrade-to-Typescript-without-anybody-noticing-part-2.md),
to learn about the various kinds of fixes.
