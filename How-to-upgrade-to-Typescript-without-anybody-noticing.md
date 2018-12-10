# How to Upgrade to TypeScript Without Anybody Noticing

This guide will show you how to upgrade to Typescript without anybody
noticing. Well, people *might* notice &mdash; what I really mean is
that you won't have to change your build at all. You'll have the
ability to get errors and completions in your editor and run `tsc`
from the command line, but you won't have to integrate Typescript into
your build.

The way to do this is to not use Typescript. Stick with Javascript and
use JSDoc to provide type information. This is less convenient than
Typescript's syntax for types, but it means that your files stay
boring old Javascript.

Here's how. I'm going to use
[typescript-eslint-parser](https://github.com/eslint/typescript-eslint-parser)
as an example package so you can follow along if you want.

## Add tsconfig

I like to start with `tsc --init` and change the settings it gives you.

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

For Javascript, your "compilerOptions" will pretty much always look
like this. "resolveJsonModule" is optional, but you'll need it if your
code ever `require`s a JSON file, so that Typescript will analyze it
too. *Officially*, strict should be false, but as you will see later,
 strict mode can be pretty great for Javascript code, not just
 Typescript code.

For typescript-eslint-parser, I added an exclude list. I wanted to
check *all* the source files, because it's nice to have high
aspirations. But it turns out the parser's tests take the form of JS
files themselves, and I don't want to check those. Most of them are
malformed in some way.

You might want to use "include" if, say, you only want to check your
source and not your tests or scripts. Or you can use "files" to give
an explicit list of files to use, but this is annoying except for
small projects.

OK, you're all set. Run tsc and make sure it prints out errors. Now open up
files in your editor and make sure the same errors show up there.

Congratulations! You've done the only *required* part. Everything else
just helps reduce the number of errors you see when editing.

## Install @types packages.

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

1. Add missing types to dependencies.
2. Fix up type annotations that refer to dependencies.
4. Add missing types in your own code.
3. Work around missing types in dependencies.
5. Fix errors in existing types.

6. Stub out modules in a d.ts. (When you don't mind checking in
Typescript code, although others don't necessarily need to edit it.)

## Add missing types in dependencies

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

## Fix references to types in dependencies

In analyze-scope.js, I see quite a few errors about missing estree
types like Identifier and ClassDeclaration. That's weird, because
those types *do* exist in estree. The problem is that they're not
imported. I'd like to write

```js
import { Identifier, ClassDeclaration } from "estree";
```

But this doesn't work because those are types, not values. The import
will fail at runtime. Instead, you need to use an *import type*. An
import type is just like a dynamic import, except that it's used as a
type. So, just like you could write:

```js
const fs = import("fs");
```

to dynamically import `fs`, you can write:

```ts
var id: import("estree").Identifier = ...
```

to import the type `Identifier` without an import statement. And,
because it's inconvenient to repeat `import` all over the place, you
usually want to write a `typedef`:

```js
/** @typedef {import("estree").Identifier} Identifier */
```

## Add missing types in your own code

Fixing these types still leaves a lot of undefined types in
analyze-scope.js. The types *look* like estree types, but they're
prefixed with TS-, like TSTypeAnnotation and TSTypeQuery. It turns out
that these types are specific to typescript-eslint-query. So we'll
have to define these types ourselves. To start, I defined a lot
more typedefs with any. This got rid of the errors:

```js
/** @typedef {any} TSTypeQuery */
// etc ...
```

Then I changed the typedefs one by one to 'unknown', and went looking
for errors that popped up:

```js
/** @typedef {unknown} TSTypeQuery */
```

Turns out that only one usage exists, in the method TSQualifiedName:

```js
    /**
    * Create reference objects for the object part. (This is `obj.prop`)
    * @param {TSTypeQuery} node The TSTypeQuery node to visit.
    * @returns {void}
    */
    TSQualifiedName(node) {
        this.visit(node.left);
    }
```

TSTypeQuery is supposed to have a left property, so I changed from
`unknown` to `{ left: unknown }`:

```js
/** @typedef {{ left: unknown }} TSTypeQuery */
```

With little knowledge of typescript-eslint-parser, I can only use the
code itself to figure out the types, but of course if you work on a
project, you'll have a better idea of what the types should be.

(and use them)
(and decide where to store them)
(CHECK YOUR WORK)

## Work around missing types

Sometimes you'll find that one of your dependencies has no `@types`
package at all. You are free to define types and contribute them to
Definitely Typed, of course, but you usually need a quick way to work
around missing dependencies. The easiest way is to add your own
typings file to hold workarounds. For example, the next error left in
analyze-scope.js is on:

```js
new PatternVisitor(this.options, node, callback)
```

It says that PatternVisitor expects 0 arguments but got 3. Why did it
expect 0? Well, PatternVisitor doesn't have its own constructor, but it
extends the class from 'eslint-scope/lib/pattern-visitor'.
Unfortunately, `@types/eslint-scope` doesn't export
`lib/pattern-visitor`, so PatternVisitor doesn't *get* the 3-parameter
constructor. It just has a default 0-parameter constructor.

You could go off and add types for all of pattern-visitor.js, but all
you really need right now is enough to get analyze-scope.js to compile
without errors. You can add the rest later. Here's what I did:

1. Create `types.d.ts` at the root of typescript-eslint-parser.
2. Add the following code:

```ts
declare module "eslint/lib/pattern-visitor" {
    class PatternVisitor {
        constructor(x: any, y: any, z: any) {
        }
    }
    export = PatternVisitor;
}
```

This declares an "ambient module", which is a weaselly name for "my
fake workaround module". It's designed for exactly this case, though,
where you are overwhelmed by the amount of work you need to do and
just want a way to reduce it for a while. You can even put multiple
`declare module`s in a single file so that all your workarounds are in
one place.

This fixes the immediate problem, but a number of errors pops up on
`PatternVisitor extends OriginalPatternVisitor`. So the definition
needs to be more complete. Here's what I ended up with:

```ts
declare module "eslint-scope/lib/pattern-visitor" {
    import { Node, Expression, SpreadElement } from "estree";
    type Options = {
        topLevel: boolean;
        rest: boolean;
        assignments: any[];
    };
    class PatternVisitor {
        constructor(
            options: any,
            rootPattern: any,
            callback: (p: any, options: Options) => void);
        rightHandNodes: Array<Expression | SpreadElement>;

        Identifier(node: Node): void;
        visit(node: Node): void;
    }
    export = PatternVisitor;
}
```

I got this by (1) looking at pattern-visitor.js and (2) the estree
types. eslint-scope uses estree types without explicitly referring to
them, so an eventual correct fix would change eslint-scope. For now, I
have enough to keep going.

## Fix errors in existing types

Unfortunately, the improved type for pattern-visitor once again causes
an error on `new PatternVisitor`. This time, the callback's type,
`Function` isn't specific enough to work with the specific function
type of the callback:

``ts
callback: (p: any, options: Options) => void
```

So the existing JSDoc type annotation needs to change:

```js
    /**
     * Override to use PatternVisitor we overrode.
     * @param {Identifier} node The Identifier node to visit.
     * @param {Object} [options] The flag to visit right-hand side nodes.
     * @param {Function} callback The callback function for left-hand side nodes.
     * @returns {void}
     */
    visitPattern(node, options, callback) {
```

I changed `@param {Function} callback` to the more precise
`@param {(p: any, options: Options) => void} callback`.

Except that Options isn't exported by my pattern-visitor workaround.
So I moved it in the global namespace, outside the `declare module`:

```ts
type Options = {
    topLevel: boolean;
    rest: boolean;
    assignments: any[];
};

declare module "eslint-scope/lib/pattern-visitor" {
    import { Node, Expression, SpreadElement } from "estree";
    class PatternVisitor {
        constructor(
            options: any,
            rootPattern: any,
            callback: (p: any, options: Options) => void);
        rightHandNodes: Array<Expression | SpreadElement>;

        Identifier(node: Node): void;
        visit(node: Node): void;
    }
    export = PatternVisitor;
}
```

This is messy but at the same time *horribly* convenient.

## Add type annotations to everything else

This *usually* means adding type annotations and types to support
them. You won't have to change the actual code much unless you find a
bug. If you want to see how far you have to go, turn on "strict" and
see how many errors you get.

You can use the infer-from-usage suggestions to help.




== Some buy-in ==
1. Add tsconfig and check it in
2. Add d.ts to hold non-installed types
3. Add a build task and maybe even hook it up to CI.
