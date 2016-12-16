# Compiler Manual

Welcome to the compiler manual. It details the compiler implementation
and its philosophy. Because it focusses on implementation, it's
necessarily out-of-date and incomplete. Also, in this version, I
totally make guesses about things I'm not sure about.

## Overview

The basic data structures used by the compiler (and elsewhere) are
**Node**, **Symbol** and **Type**.

### Node

The parser produces nodes in the form of an Abstract Syntax Tree
(AST). Nodes track syntax kind, location, and children. For example,
`a + 1` produces a BinaryExpression with three children: an
Identifier(a), a PlusToken and a LiteralExpression(1).

### Symbol

The binder produces nodes in the form of a symbol table. A symbol
tracks the declaration(s) of an identifier. The declaration is a
**Node**. For example, in `let a = 1`, the binder creates a symbol for
`a`, which future uses of `a` can then look up in a symbol table.
accessibility.

### Type

TODO: Write this.

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
everything, even things that exist in Typescript, into completely
separate AST nodes like JSDocArrayType for parsing `T[]`. There are a
few exceptions to this: type literals call back into the normal
parser.


JSDoc is stored in the **jsDocComments** property of the node that
it's associated with. Each JSDoc then contains a top-level **comment**
and a list of **tags**. There are various kinds of tags, but they all
have an associated string **comment** as well. For example:

```js
/**
 * simple function
 * @returns nothing
 */
function f() { }
```

Attaches a JSDoc to `f` with `comment = "simple function"` and a
JSDocReturnTag that has `comment = "nothing"`.

Since JSDoc supports non-local annotations &mdash; parameter
information usually occurs on a `@param` tag on the containing
function, for example &mdash; retrieval becomes complicated.
Typescript chooses to stitch the information back together when it's
needed, usually while the checker is running. More on that in the
checker section.

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

### Type Inference

Type inference is basically the hairiest thing the compiler does. Here
are some notes from my latest brush with it; I make no claim that
these are complete, just useful.

Type inference is a triple-nested loop. The outer loop checks
overloads in order. The middle loop initially ignores
context-sensitive parameters, then enables them one at a time. And the
inner loop infers from parameters one at a time (skipping context-sensitive
parameters that haven't been enabled yet).

The outer two loops are in `chooseOverload`, called from inside
`resolveCall` and the inner loop is in `inferTypeArguments`. The
actual inference rules are in `inferFromTypes` and are pretty
straightforward.

#### Contextual Typing

Contextual typing is especially tricky because it can **fix** type
parameters. When the compiler decides that a parameter is contextually
typed and tries to determine the contextual type, it *must* also fix the type
parameters. That's because the contextual type needs to be the
instantiated type. But getting the instantiated type means that the
compiler has to know how to map the type parameter to the instantiated
type &mdash; which is exactly the task of type inference!

What's more, neither assigning a contextual type nor fixing a type
parameter is reversible. So once the contextual type has forced us to
fix a type parameter, we're stuck with both decisions.

That's why contextual types are always processed last; the other
parameters already have types, so type inference is not destructive.
But type inference of contextually typed things actually causes the
types to be added to them **permanently**.

#### Notes

* The inner loop (parameters) actually always infers from normal
parameters first. Then it infers from context-sensitive parameters,
but only the ones that have been enabled in the middle loop.

* The outer loop (overloads) throws away the inference context each
time. But the type of parameters may be set if they are context
sensitive. This is occasionally, unfortunately observable.


### JSDoc

JSDoc is handled half-heartedly in the checker as well. That's largely
for historical reasons; the checker didn't care about JSDoc at all
until Typescript was able to handle Javascript because it didn't
contribute types or even symbols to the compilation. (That's right:
JSDoc types in Typescript are *ignored*. It only uses the documentation.)

That's different now because JSDoc is the only way to specify types in
Javascript. But the checker still handles it in a very ad-hoc way
&mdash; everywhere you might get some information from a type
annotation, you also need to remember to call `getJSDocType` if you
are checking a Javascript file.

Alas, since JSDoc stores information non-locally, retrieving type
information for a specific node is not that easy. The worst example is
parameters: type information can be stored as a comment for the
parameter, for the function or for the variable that the function is
assigned to:

```js
var /** @param {number} x - a */f =
  /** @param {number} x - b */ function (/** c */x) {
  };
```

Typescript delays the work needed to stitch together this information
until it's needed (as usual), hoping that (1) it won't be needed and
(2) it won't be that hard to retrieve in the common case and (3) that
once cached it will be needed again.

Fortunately, `getJSDocType` conceals this work from you
pretty well. If you're in the checker, that's probably all you'll
need, except perhaps related functions like `getJSDocReturnTag` (to
get the return type of a Javascript function) or `getJSDocTemplateTag`
(to get the type parameters of a Javascript function).

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

#### Literal widening

Literal widening is different from nullable widening. It
doesn't happen in the same place, and it widens a subtype (the literal
type) to a supertype (string, number or boolean). Nullable widening
widens `undefined` and `null` to `any`. It probably wouldn't exist if
`null` and `undefined` had been denotable types before 2.0.

You can widen literals using two different functions,
`getWidenedLiteralType` and `getBaseTypeOfLiteralType`.

The difference is that `getBaseTypeOfLiteralType` unconditionally
widens whereas `getWidenedLiteralType` only widens if the literal type
is fresh. For full discussion of freshness,
[see the PR that implemented it](https://github.com/Microsoft/TypeScript/pull/10676).

Briefly, literal types start fresh if they arise from a literal
expression. Fresh literal types widen when they are assigned to a
non-cost/readonly location. Non-fresh literal types do not widen. For
example,

```ts
const fresh = 1;
type notFresh = 1;
let f = fresh; // f is mutable, so '1' widens to 'number'
let i: notFresh = 1; // even though i is mutable, it has type '1'
```

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
