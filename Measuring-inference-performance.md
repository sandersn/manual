<!--
# Overview

When I'm working on Typescript's infer-from-usage codefix, which looks
at usage of variable to infer their types, I need to make sure that
I'm making progress on an ill-defined problem with many aspects.
Because of the ad-hoc nature of the inference, it's easy to improve
one aspect while making another worse. I need to know whether I'm
improving the system as a whole.

To do that, I need to

1. Compare performance of a change against performance in the past.
2. Look at the effect of a change across multiple styles of
Javascript.

To do that, I created the set of scripts at sandersn/measure. These
scripts

1. Check out a particular commit of Typescript and build it.
2. Run the infer-from-usage codefix on Typescript's user tests in a
   *different* clone of Typescript. 
3. Compare before/after: the number of `any`s on "declaration sites"
   -- nodes that are expected to have a type. Some nodes in the tree
   don't have a reasonable type, like the keywords `import` or `if`, for
   example.
4. Compare before/after: the number of compile errors.

The counts of `any` and errors allow me to roughly track performance
over time. And since the script leaves the second clone of Typescript
in the post-codefix state, I can inspect the user tests to learn how
the codefix works on actual code. The combination of overall
performance combined with ability to inspect individual changes lets
me make progress on a problem with a defined end-goal.
-->

# Implementation

## Bookkeeping (function main)

refactorAll.js has two outputs:

1. The codefixed programs.
2. A JSON file tracking how many `any` types and how many `errors` for
each program before and after running the infer-from-usage codefix.

For the second, I read some data from JSON on startup. And I have a
small function that counts the number of `any` by walking the tree and
asking for the type of interesting nodes. Most people will only care
about the codefix that happens in the main loop.

## Reset state (function resetUserTest)

My state is complicated and uninteresting. Yours is probably much
simpler; if so, you should be able to skip this section with a
`git checkout -- .`

In my code, I need to:

1. Check out a particular commit of Typescript and build it.
2. Either `npm install` or `git clone` each test program.

That's what `resetUserTest` does, with some extras. It's a simplified clone of code from
Typescript's external test runner.

## Main Loop (function codeFix)

For each project, I need to do some initial setup for the language
service.

1. Parse a tsconfig. I manually set noImplicitAny so that implicit
anys count as errors.
2. Set up initial file state, which is just a map from file names to ids
that increment on each change to the file.
3. Make a simple LanguageServiceHost that mostly delegates to `ts.sys`
to read from the filesystem. I got this example from the Typescript
Wiki.
4. Create a language service, which actually does the refactor.
5. Create a program, which reports errors and types.

Then I count anys/errors, run the refactor and count again.

### Count anys

For each file in the program, I walk the tree: I call
`getTypeAtLocation` for every expression, identifier and declaration
name and record the number of types that are `any`.

### Count errors

This is easy: I call `getPreEmitDiagnostics` to get errors and record the number.

### Run the refactor

For each file in the program, I call `getCombinedCodeFix` with
`"inferFromUsage" to get a list of text changes that need to be made.
Then, very inefficiently, for each change, I read the file from disk,
call `textChanges.applyChanges` and write the file to disk again.
You could likely speed this up by reading/writing outside the loop.
