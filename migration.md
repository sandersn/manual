# Improving Typescript Migration #

When migrating code for Typescript you generally have to worry about
three categories of code: the new code you want to write, the old code
you already have, and the dependencies for your project.

I would like to further improve the Typescript migration process.
Based on the 3-part breakdown of old/new/dependency, and on a survey
and responses on twitter, I think the two biggest problems are:

1. Getting types for dependencies; they can be incorrect, out-of-date or missing.
2. Adding types to tricky Javascript code.

From recent responses on twitter, I found that both problems were
about equally common. This agrees with survey data for checkJS from
last year, which found that users were generally happy with Javascript
features.

Based on this, I propose that we improve handling of dependency's
types, and research what could be done to add types to complex
Javascript. I'll cover two kinds of dependency problems:

## Bad Types ##

Types can be bad for several reasons: they could be
1. incomplete
2. for the wrong version of the package
3. incorrect

A user finds out that types are bad by seeing errors in code that
consumes the package. We assume the user's code is correct since this
is a migration scenario, and can use that code to figure out the
correct fix to the bad types.

We should provide a tool that
1. Diagnoses which problem you have.
2. If the version is wrong, it installs the correct version.
3. If the types are partly incorrect, it clones the types into your
   repo and overrides those types.
4. If the types are partly missing, it does the same thing. It can
   also use the package's Javascript source to infer types.

## Missing Types ##

Missing types are similar to partly missing types above. We can use
the user's code to infer what the types should be, as well as look at
the package's source code.

We should provide a tool that
1. checks for types on Definitely Typed. (with version matching as above)
2. creates types based on the original library + your consuming code.

In both scenarios, if you end up with a local typing in your
repository, the tool should allow you to submit the typings to
Definitely Typed or to the package owner.

## Implementation ##

Here are the large-ish features needed for this tool.

1. d.ts from usage
2. Better understanding of UMD/AMD -- needed for npm-shipped JS.
3. d.ts from JS
4. compare existing d.ts to usage
5. Swap multiple versions looking for least errors (based on (4))
6. Drop files in correct override location.
7. Create/copy files into correct DT location.

Step (3) is a hard, but valuable, problem to solve completely and
correctly. It would also be useful for incrementally building projects
that contain JS.

## JS to TS conversion ##

At the same time, we should research file-by-file conversion to Typescript:

1. Choose several repos, based on similar criteria to the user
  tests.
2. Add tsconfig etc to get the project building with allowJs.
3. Progressively convert files, classifying errors along the way:
  a. Before running whole-file codefixes.
  b. After running whole-file codefixes.
4. Check whether Typestat's additional fixes address real problems.

This will tell us how difficult the problem currently is -- whether a
machine could feasibly fix the problems that remain after running our
current codefixes.

# Appendix A: Analysis of replies on Twitter #

## Types ##

- Missing types for dependencies (or own dependencies): 6
- Bad types from dependencies: 7
- Hard add complex types to old code: 1

## Errors ##

- bad/dynamic js: 11
- implicit any: 1
- complex types: 1

## Documentation ##

- No file-by-file tutorial: 1
- export=/import = require: 2
    twitter suggests: support const foo = require/module.exports =
- Time needed to teach Typescript: 2
- Types too complex to understand: 1

## Other ##

- react: 3
    especially higher order components
- build: 3
    one each from closure, babel and webpack
- tests with ts-node: 3
    twitter suggests: ts-jest


Other subjects: create-react-app, globals, classes (?)

resistance from others: 1
