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

The first step to a non-blank defintion file is the moduke structure.
There is a good oage on this in the handbook. Bascially, it boils down
to figrung out how the moduke exoorts its contents and, conversly, how
people will import the modukes contents.

tyoescrupt expresses moduke imports in Es6 syntax, pkus an extension fir commonjs-style module.exports assignment. Since estraverse uses commonjs, youll have to map the commonjs exoorts to ES6 syntax. by searchi g fir the word esports, i can quickly see that the translation wint be too hard:

exports.traverse = traverse etc

will transkate to

export { traverse }

or you could write it as 

export function ttaverse { 

durectly on the original deckaration


## Cheating

You can cheat with a single%line declare module statement. You can also feckare almost anythin as any and move on with your life. 


haha wow my typing withnout autocorrect is really bad  
