This guide will show you how to fix Typescript compile errors in
Javascript project that recently added Typescript support via a
`tsconfig.json`. It assumes that the `tsconfig.json` is configured
according to
[the description in part 1 of this post](How-to-upgrade-to-Typescript-without-anybody-noticing-part-1.md),
and that you also installed types for some of your dependencies from
the `@types/*` namespace. This guide is more of a list of tasks that
you can pick and choose from, depending on what you want to fix first.
Here's are the tasks:

1. Add missing types in dependencies.
2. Fix references to types in dependencies.
4. Add missing types in your own code.
3. Work around missing types in dependencies.
5. Fix errors in existing types.
6. Add type annotations to everything else.

This guide does *not* teach you how to write type definitions.
[The Declaration section of the Typescript handbook](http://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html)
is the best place to learn about that. Here, you'll just see types
presented without a lot of explanation.


## Add missing types in dependencies

Let's start with @types/shelljs. In Makefile.js, I see a few errors.
The first is that the module `require('shelljs/make')` isn't found.
The second group of errors is that the names 'find', 'echo' and a few
others aren't found.

These errors are related. It turns out that `@types/shelljs` doesn't
even include shelljs/make.d.ts right now. It's completely missing. If
you
[look at the source](https://github.com/shelljs/shelljs/blob/master/make.js),
shelljs/make.js does two things:

1. Add the contents of the parent shelljs to the global scope.
2. Add a global object named 'target' that allows you to add make
targets.

Let's say you want to add make to Definitely Typed so that it is
available in `@types/shelljs`. Your first step is to create make.d.ts
inside node_modules/@types/shelljs/. This is the wrong location
&mdash; it's inside your own node_modules folder &mdash; but it makes
development super easy to test that you're actually fixing the
missing stuff. You can create a proper PR after everything is working.

Start with this:

```ts
import * as shelljs from './';
declare global {
  const cd: typeof shelljs.cd;
  const pwd: typeof shelljs.pwd;
  // ... all the rest ...
}
```

This copies all of shelljs' contents into the global namespace.
Then add the type for 'target' to the globals as well:

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

This exposes a couple more errors. See the section on fixing errors in
existing types for how to fix those.

Now we want to publish this to Definitely Typed:

1. Fork DefinitelyTyped on github.
2. `git clone https://github.com/your-name-here/DefinitelyTyped`
3. `cp node_modules/@types/shelljs/make.d.ts ~/DefinitelyTyped/types/shelljs/`
4. `git checkout -b add-shelljs-make`
5. Commit the change and push it to your github fork.
6. Create a PR for the change.

If there are lint problems, the CI run on Definitely Typed will catch them.

For more detail on writing definitions for Definitely Typed, [see the
Declaration section of the Typescript handbook](http://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html).

## Fix references to types in dependencies

typescript-eslint-parser actually has quite a bit of type information
in its source already. It just happens to be written in JSDoc, and
it's often almost, but not quite, what Typescript expects to see. For
example, in analyze-scope.js, `visitPattern` has an interesting mix of
types:

```js
/**
 * Override to use PatternVisitor we overrode.
 * @param {Identifier} node The Identifier node to visit.
 * @param {Object} [options] The flag to visit right-hand side nodes.
 * @param {Function} callback The callback function for left-hand side nodes.
 * @returns {void}
 */
visitPattern(node, options, callback) {
    if (!node) {
        return;
    }

    if (typeof options === "function") {
        callback = options;
        options = { processRightHandNodes: false };
    }

    const visitor = new PatternVisitor(this.options, node, callback);
    visitor.visit(node);

    if (options.processRightHandNodes) {
        visitor.rightHandNodes.forEach(this.visit, this);
    }
}
```

In the JSDoc at the start, there's an error on `Identifier`. (`Object`
and `Function` are fine, although you could write more specific
types.) That's weird, because those types *do* exist in estree. The
problem is that they're not imported. Typescript lets you import types
directly, like this:

```js
import { Identifier, ClassDeclaration } from "estree";
```

But this doesn't work in Javascript because those are types, not
values. Types don't exist at runtime, so the import will fail at
runtime when `Identifier` is not found. Instead, you need to use an
*import type*. An import type is just like a dynamic import, except
that it's used as a type. So, just like you could write:

```js
const estree = import("estree);
```

to dynamically import the `Identifier` type from estree, you can write:

```js
/** @type {import("estree").Identifier */
var id = ...
```

to import the type `Identifier` without an import statement. And,
because it's inconvenient to repeat `import` all over the place, you
usually want to write a `typedef` at the top of the file:

```js
/** @typedef {import("estree").Identifier} Identifier */
```

With that alias in place, references to `Identifier` resolve to the
type from estree:

```js
/**
 * @param {Identifier} node now has the correct type
 */
```

[Here's the commit.](https://github.com/eslint/typescript-eslint-parser/commit/f02b62d70cbabeebcfb6cd75dfaa2d94d0679fd5)

## Add missing types in your own code

Fixing these types still leaves a lot of undefined types in
analyze-scope.js. The types *look* like estree types, but they're
prefixed with TS-, like `TSTypeAnnotation` and `TSTypeQuery`. Here's where
`TSTypeQuery` is used:

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

It turns out that these types are specific to typescript-eslint-query.
So you'll have to define them yourself. To start, define the typedefs
you need as `any`. This gets rid of the errors at the cost of accuracy:

```js
/** @typedef {any} TSTypeQuery */
// lots more typedefs ...
```

At this point, you have two options: bottom-up discovery of how the
types are used, or top-down documentation of what the types should be.

Bottom-up discovery, which is what I'll show below, has the advantage
that you will end up with zero compile errors afterward. But it
doesn't scale well; when a type is used throughout a large project,
the chances of it being *mis*used are pretty high.

Top-down documentation works well for large projects that already have
some kind of documentation. You just need to know how to translate
documentation into Typescript types;
[the Declaration section of the Typescript handbook](http://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html)
is a good starting point for this. You will sometimes have to change
your code to fit the type using the top-down approach as well. Most of
the time that's because the code is questionable and needs to be
changed, but sometimes the code is fine and the compiler gets confused
and has to be placated.

Let's use bottom-up discovery in this case because it looks
like top-down documentation would involve copying the entire
Typescript node API into analyze-scope.js. To do this, change the
typedefs one by one to 'unknown', and look for errors that
pop up. For example:

```js
/** @typedef {unknown} TSTypeQuery */
```

Now there's an error is on the usage of `TSTypeQuery` in
`TSQualifiedName`, `node.left`:

```js
/**
 * Create reference objects for the object part. (This is `obj.prop`)
 * @param {TSTypeQuery} node The TSTypeQuery node to visit.
 * @returns {void}
 */
TSQualifiedName(node) {
    this.visit(node.left);
    //              ~~~~
    // error: type 'unknown' has no property 'left'
}
```

Looks like TSTypeQuery is supposed to have a left property, so change
`TSTypeQuery` from `unknown` to `{ left: unknown }`. There's no more
indication of what the type of `left` is, so leave it as
`unknown`:

```js
/** @typedef {{ left: unknown }} TSTypeQuery */
```

As you can see, bottom-up type discovery can be a bit unsatisfying and
underspecified, but it's less disruptive to existing code.
[Here's the commit.](https://github.com/eslint/typescript-eslint-parser/commit/57b517c5a763ee47e92aee93d0b97ed096eaeec7)

## Work around missing types

Sometimes you'll find that one of your dependencies has no `@types`
package at all. You are free to define types and contribute them to
[Definitely Typed](https://github.com/DefinitelyTyped/DefinitelyTyped),
of course, but you usually need a quick way to work around missing
dependencies. The easiest way is to add your own typings file to hold
workarounds. Let's look at `visitPattern` in analyze-scope.js again:

```js
/**
 * Override to use PatternVisitor we overrode.
 * @param {Identifier} node The Identifier node to visit.
 * @param {Object} [options] The flag to visit right-hand side nodes.
 * @param {Function} callback The callback function for left-hand side nodes.
 * @returns {void}
 */
visitPattern(node, options, callback) {
    if (!node) {
        return;
    }

    if (typeof options === "function") {
        callback = options;
        options = { processRightHandNodes: false };
    }

    const visitor = new PatternVisitor(this.options, node, callback);
    visitor.visit(node);

    if (options.processRightHandNodes) {
        visitor.rightHandNodes.forEach(this.visit, this);
    }
}
```

Now there is an error on

```js
const visitor = new PatternVisitor(this.options, node, callback)
                ~~~~~~~~~~~~~~~~~~
                Expected 0 arguments, but got 3.
```

But if you look at
[`PatternVisitor` in the same file](https://github.com/sandersn/typescript-eslint-parser/blob/master/analyze-scope.js#L38),
it doesn't even *have* a constructor. But it does extend
`OriginalPatternVisitor`:

```js
const OriginalPatternVisitor = require("eslint-scope/lib/pattern-visitor");
// much later in the code...
class PatternVisitor extends OriginalPatternVisitor {
    // more code below ...
}
```

Probably `OriginalPatternVisitor` has a 3-parameter constructor which
`PatternVisitor` inherits. Unfortunately, `eslint-scope` doesn't
export `lib/pattern-visitor`, so PatternVisitor doesn't *get* the
3-parameter constructor. It ends up with a default 0-parameter
constructor.

As described in "Add missing types in dependencies", you could add
`OriginalPatternVisitor` in `lib/pattern-visitor.d.ts`, just like we
did for `make.d.ts` in shelljs. But when you're just getting started,
sometimes you just want to put a temporary type in place. You can add
the real thing later. Here's what you can do:

1. Create `types.d.ts` at the root of typescript-eslint-parser.
2. Add the following code:

```ts
declare module "eslint/lib/pattern-visitor" {
    class OriginalPatternVisitor {
        constructor(x: any, y: any, z: any) {
        }
    }
    export = OriginalPatternVisitor;
}
```

This declares an "ambient module", which is a pompous name for "my
fake workaround module". It's designed for exactly this case, though,
where you are overwhelmed by the amount of work you need to do and
just want a way to fake it for a while. You can even put multiple
`declare module`s in a single file so that all your workarounds are in
one place.

After this, you can improve the type of OriginalPatternVisitor in the
same bottom-up or top-down way that you would improve any other types.
For example, you can look at
[pattern-visitor.js in eslint](https://github.com/eslint/eslint-scope/blob/master/lib/pattern-visitor.js#L40)
to find the names of the constructor parameters. Then, a little lower
in the
[`Identifier` method of `OriginalPatternVisitor`](https://github.com/eslint/eslint-scope/blob/master/lib/pattern-visitor.js#L66)
there is a usage of `callback` that gives enough information to guess
its type.

[Here's what you'll end up with](https://github.com/eslint/typescript-eslint-parser/commit/cd00d200049e449d395d6fe8d480c4620994f225):

```ts
declare module "eslint-scope/lib/pattern-visitor" {
    import { Node } from "estree";
    type Options = unknown;
    class OriginalPatternVisitor {
        constructor(
            options: Options,
            rootPattern: Node,
            callback: (pattern: Node, options: Options) => void);
    }
    export = OriginalPatternVisitor;
}
```

## Fix errors in existing types

Unfortunately, the improved type for pattern-visitor once again causes
an error on `new PatternVisitor`. This time, the callback's type,
`Function` isn't specific enough to work with the specific function
type of the callback:

```ts
callback: (pattern: Node, options: Options) => void
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

The right fix is to chang the type `Function` to the more precise
`(pattern: Node, options: Options) => void`:

```js
/**
 * Override to use PatternVisitor we overrode.
 * @param {Identifier} node The Identifier node to visit.
 * @param {Object} [options] The flag to visit right-hand side nodes.
 * @param {(pattern: Node, options: import("eslint-scope/lib/pattern-visitor").Options) => void} callback The callback function for left-hand side nodes.
 * @returns {void}
 */
visitPattern(node, options, callback) {
```

## Add JSDoc types to everything else

Once you get all the existing type annotations working, the next step
is to add JSDoc types to everything else. You can turn on
`"strict": true` to see how far you have to go &mdash; among other things,
this marks any variables that have the type `any` with an error.

You should fall into a back-and-forth of adding new JSDoc type
annotations and fixing old types. Usually old types just need to be
updated to work with Typescript, but sometimes you'll find a bug.
