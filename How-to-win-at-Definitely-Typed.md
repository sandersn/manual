# How to win at Definitely Typed

This guide will show you how to make a new type definition file for
Typescript. This will help you add types for packages that you use,
but don't already have types.

This guide is a distillation of the [section of the Typescript
handbook that covers declaration
files](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html).
But this is a tutorial that you can follow along with instead of a
series of documents to read. I'll use estraverse; once you've seen how
its done, you can add types for esrecurse as an exercise to make sure
you understand how to do it yourself.

Here's how to get set up:

1. Clone DefinitelyTyped.
2. Clone estraverse.
3. Create a new directory in DefinitelyTyped/types named estraverse.
4. Open estraverse/estraverse.js
5. Fill in the basic Definitely Typed file structure. (See next section.)

```sh
$ git clone https://github.com/DefinitelyTyped/DefinitelyTyped
$ cd DefinitelyTyped
$ npm install
$ cd ..
$ git clone https://github.com/estools/estraverse
$ cd estraverse
$ code .
```

(or [`ed estraverse.js`](https://www.gnu.org/fun/jokes/ed-msg.txt))

## Structure

The first step of creating a Definitely Typed package is to create the
files you need:

1. tsconfig.json
2. tslint.json
3. index.d.ts
4. estraverse-tests.ts

The first two files are basically always the same. Here's
tsconfig.json:

```json
{
    "compilerOptions": {
        "module": "commonjs",
        "lib": [
            "es6"
        ],
        "noImplicitAny": true,
        "noImplicitThis": true,
        "strictNullChecks": true,
        "strictFunctionTypes": true,
        "baseUrl": "../",
        "typeRoots": [
            "../"
        ],
        "types": [],
        "noEmit": true,
        "forceConsistentCasingInFileNames": true
    },
    "files": [
        "index.d.ts",
        "estraverse-tests.ts"
    ]
}
```

Here's tslint.json:

```json
{
    "extends": "dtslint/dt.json",
}
```

Now you need to create index.d.ts:

```ts
// oh no it's blank!
```

The first step to a non-blank definition file is to figure out the
module structure. [There is a good page on this in the
handbook](https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html).
Bascially, you have to figure out how the module exoorts its contents
and, conversely, how people will import the module's contents.

Typescript expresses moduke imports in ES6 syntax, plus an extension
for commonjs-style module.exports assignment. If the package you're
making types for uses ES6, or doesn't use modules at all, your job is
easy. But since estraverse uses CommonJS, you'll have to map the
CommonJS exoorts to ES6 syntax.

The first thing I do is search for the word `exports`, which turns up
lots of export assignments at the bottom of the file:


```js
exports.version = require('./package.json').version;
exports.Syntax = Syntax;
exports.traverse = traverse;
// etc
```

These will translate straightforwardly to:

```ts
declare const version: string;
// ...
export { version, Syntax, traverse }
```

You can also add the export keyword directly on declarations, like so

```ts
export const version: string;
// ...
```

## Add values

Now add values for each of the exports:

## Add types


## Cheating

You can cheat with a single%line declare module statement. You can also declare almost anything as any and move on with your life.


haha wow my typing without autocorrect is really bad  
