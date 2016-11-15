# Compiler Manual

Welcome to the compiler manual. It details the compiler implementation
and its philosophy. Because it focusses on implementation, it's
necessarily out-of-date and incomplete. Also, in this version, I
totally make guesses about things I'm not sure about.

## Parser

It's a recursive descent parser. It's pretty resilient, so if you
search for functions matching the thing you want to change, you can
probably get away with just adding the code to parse. There aren't any
surprises in the general implementation style here.

TODO: More here later.

### JSDoc

The parser parses JSDoc half-heartedly. You have to explicitly call
`addJSDocComment` after calling `finishNode` &mdash; it's not part of
`finishNode` itself because so many nodes can not have JSDoc on them.

Then once it starts parsing it uses a completely different tokeniser,
`nextJSDocToken()`, from the normal parser. (It doesn't ignore
whitespace and returns way fewer SyntaxKinds). And it parses
everything, even things that exist in typescript, into completely
separate AST nodes like JSDocReturnTag. There are a few exceptions to
this: type literals call back into the normal parser.

JSDoc storage is simple right now; perhaps too simple. A JSDoc comment
is put on the node it occurred before. Since JSDoc supports non-local
annotations, retrieval becomes complicated. Typescript chooses to
stitch the information back together when it's needed, usually while
the checker is running. More on that in the checker section.

## Binder

It gathers symbols from files. There's one created when you start the
compiler, and that's it. It creates symbols for
declarations so that the checker can stick types on them.

TODO: This is woefully incomplete.

TODO: It also sets up some data structures that the checker uses, like
control flow.

TODO: It also marks nodes with their language for the emitter. For
example, all types are typescript syntax, exponentiation is ES2016
syntax and `class` is ES2015 syntax.

## Checker

The checker makes sure the types are right. 'Nuff said!

Actually, it's 20,000 lines long, and does almost everything that's
not syntactic. Surprisingly, a checker gets created every time the
language service requests information because it tries to present an
immutable interface. This works because the checker is very lazy.

### JSDoc

JSDoc is handled half-heartedly in the checker as well. That's largely
for historical reasons; the checker didn't care about JSDoc at all
until Typescript was able to handle Javascript because it didn't
contribute types or even symbols to the compilation. (That's right:
JSDoc types in Typescript are *ignored*. It only uses the documentation.)

That's different now because JSDoc is the only way to specify types in
Javascript. But the checker still handles it in a very ad-hoc way
&mdash; everywhere you might get some information from a type
annotation, you also need to remember to check for JSDoc.

Also, since JSDoc stores information non-locally, retrieving type
information for a specific node is not that easy. The worst example is
parameters: type information can be stored as a comment for the
parameter, for the function or for the variable that the function is
assigned to:

```ts
var /** @param {number} x - a */f =
  /** @param {number} x - b */ function (/** c */x) {
  };
```

Typescript delays the work needed to stitch together this information
until it's needed (as usual), hoping that (1) it won't be needed and
(2) it won't be that hard to retrieve in the common case. Usually I
add that (3) it will be cached, but I don't that's the case for JSDoc
right now.

### Trivia

#### Optionality

Optionality is represented two different ways. The original way is a
flag on `Symbol`. The second way is as a union type that includes
`undefined`.

If you are running with `strictNullChecks: true` then adding '?' after
a property does two things: it marks the symbol as optional and it
adds `| undefined` to the type of the property. You have to remember
to do these two things yourself whenever you synthesise a property
inside the compiler.

Conversely, the compiler actually throws away `| undefined` if
`strictNullChecks: false`. So you can't use *just* `maybeTypeOfKind(t,
TypeFlags.Undefined)` to check for optionality. You also have to check
`symbol.flags & SymbolFlags.Optional`.

## Transformer

The transformer recently replaced the emitter. It replaces the
*emitter*, which translated TypeScript to JavaScript. The
*transformer* transforms TypeScript or JavaScript (various versions)
to JavaScript (various versions) using various module systems. The
input and output are basically both trees from the same AST type, just
using different features.

The emitter is then a fairly small AST printer that can print
any TypeScript AST.

### Overview

Since the binder sets up information for the transformer since it
makes a complete pass over the AST anyway. Primarily, it marks
individual nodes with their associated dialect. In the example below,
`...x` is marked as ES2015, and `any[]` and `void` are marked as
Typescript.

```ts
function f(...x: any[]): void {
}
```

Then each transformer runs on the AST. The Typescript transformer
mostly just throws away annotations. The ES2015 transformation
actually has to emit ES5 code that implements the rest parameter.
The transforms progressively convert Typescript into older and older
Javascript dialects. If the target is something newer than ES3, then
the transforms just stop running.

TODO: This leaves out module transformers.

## Services

I don't know anything about services. And besides, this readme is for
the compiler.

## Other files

This is an unsorted inventory. Some of these deserve an in-depth
explanation, or at least a big overview of what they mean.

1. core
2. utilities
3. program
4. scanner
5. sourcemap
6. sys
7. tsc
8. types
8. commandLineParser
9. declarationEmitter
