# Parsing Hash in Typescript

## Present Private

Currently, the # character is only used two ways in Typescript. It's
the first character of private identifiers (`#private`), and it's the punctuation
in JSDoc member names (`Class#property`). Because `#private` support
predates `Class#property` support, the scanner and parser primarily
support it. Here's how it works:

1. The scanner sees a # and goes on to read an identifier. It yields a
   PrivateIdentifier token with text "#foo".
2. The parser creates a PrivateIdentifier node for the
   PrivateIdentifier token.
3. The rest of the compiler treats Identifier and PrivateIdentifier as
   the same most of the time.

There's also a special check at the beginning of scanning for the
shebang `#!`, but it's not relevant for the main scanning loop.

`Class#property` parsing works around this tight coupling of `#` with
identifiers by adding a `rescanPrivateIdentifier` function to the
scanner, which rewinds the scanner and yields a `HashToken` followed
by an `Identifier` instead of a `PrivateIdentifier`.

## Future Hash

Tightly coupling `#` with private identifiers will cause problems in
the future; the lack of remaining punctuation means that [at least one
proposal](https://github.com/tc39/proposal-record-tuple) already uses
it, and it's been proposed for [Hack-style
pipelines](https://github.com/tc39/proposal-pipeline-operator) as well.

It would be better to add `HashToken` to the normal range of the
scanner and make `PrivateIdentifier` less of a default.

## Possibilities

### Rescan PrivateIdentifier from Hash

I tried making the scanner yield `HashToken` for all non-shebang
hashes. I also added `rescanHash` to make the scanner back up and
yield a private identifier token instead.

That means I had to update the private identifier parts of the parser
to call `rescanHash`. There were more than I thought; a few central
edge detection checks in the parser want to know whether the next
token is a private identifier. For the prototype, I just hacked in a
call to `rescanHash` at the central check, but a real change would
need to make a decision at the leaves of the parser about whether it
really expects a private identifier and *only* a private identifier
whenever it sees `#`.


### Parse PrivateIdentifier as Hash + Identifier

An even bigger change, which I didn't try, is to parse
PrivateIdentifier as `Hash` + `Identifier`. I think this makes the
most sense if `#{}` and `#[]` are added to Javascript;
there will need to be a "`parseModernJSExpression`" that parses a `#`
and then decides which node to create based on whether the following
token is `Identifier`, `LeftBrace` or `LeftBracket`. However, it will
require changing the binder name-mangling code and various points in
the emitter.
