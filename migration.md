When migrating code for Typescript you generally have to worry about
three categories of code: the new code you want to write, the old code
you already have, and the dependencies for your project.

I would like to further improve the Typescript migration process.
Based on the 3-part breakdown of old/new/dependency, and on a survey
and responses on twitter, I think the two biggest problems are:

1. Getting types for dependencies; they can be incorrect, out-of-date or missing.
2. Adding types to tricky Javascript code.

From recent responses on twitter, 

Answer: it is about the same -- people have lots of bad JS that takes
a while to convert, but they also have lots of missing dependencies or
just bad types from elsewhere. Our documentation isn't great either.

2. What to do about old code? Convert it to TS or cover it with a d.ts?

Answer: Mostly people want to convert it to TS. They would prefer good
types but at least want some types. Splattering any everywhere would
not be viewed as a solution, just a TODO from the tool. And they
already HAVE the errors when they turn on noImplicitAny, so we're not
doing a huge favour there. It's unlikely that we can write a converter
that is 3-4x smarter than a Typescript novice.

Full-file type inference codefixing is ... an OK idea. It speeds up
the task, but that's not what people have complained about --
especially since any-as-default works pretty well already.

Fix two problems:

1. Bad types
  When you get bad types, run this tool.
  You give it your consuming code that you THINK is correct.
  It will
  1. check different versions for matches to your API if it's on DT
  2. override the types based on your consuming code
  3. let you run the tool again to submit the update to DT (or non-DT package)
2. Missing types
  When you are missing types, run this tool.
  You give it your consuming code.
  It will.
  1. check for DT types. (with version matching as above)
  2. create types based on the original library + your consuming code.
  3. let you run the tool again to submit the new types to DT (or non-DT package??)

Implementation:

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

Current ideas:


* Typestat upgrade
  * polish unfinished parts
  * make sure whole-file inference works correctly
* dts-gen upgrade
  * fix workflow for "missing dependency"
  * checks EXISTING types
  * checks EXISTING types
* Definitely Typed
  * process improvements
  * documentation improvements
* 3.5-to-3.6 (new or upgrade to Typestat)
* React: ??? -- Andrew is thinking about this
* Build: ???
* Just a solid way to make sure the IDE can run all relevant codefixes
  on renaming JS -> TS (and make sure all the JS-specific features
  have codefixes, and least some do not I think)

Analysis of replies on Twitter:

## Types ##

Missing types for dependencies (or own dependencies): 6
Bad types from dependencies: 7
Hard add complex types to old code: 1

## Errors ##

bad/dynamic js: 11
implicit any: 1
complex types: 1

## Documentation ##

No file-by-file tutorial: 1
export=/import = require: 2
  twitter suggests: support const foo = require/module.exports =
Time needed to teach Typescript: 2
Types too complex to understand: 1

## Other ##

react: 3
  especially higher order components
build: 3
  one each from closure, babel and webpack
tests with ts-node: 3
  twitter suggests: ts-jest


Other subjects: create-react-app, globals, classes (?)

resistance from others: 1
