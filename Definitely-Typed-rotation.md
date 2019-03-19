# What Definitely Not To Do To Definitely Typed To Do Well

## Basics

- Look at the project board.
  - Already-approved Changes
  - New Definitions
  - *Other*
- Look at PRs and merge them.
  - Already-approved changes need a cursory look.
  - New packages need a review, but see below for what kind of review.
- Watch the gitter channel.
  - Contact Nathan about technical problems.
  - Ignore everything else unless you're bored.
- Watch the status badges on the front page.
  - Contact Nathan about failures.

## Philosophy

- You need to merge over 120 pull requests in one week.
  - A normal code review accounts for social and technical factors.
  - Social:
    - Do I trust the author?
    - Do I trust the author's technical ability?
    - Do I believe the behaviour change is a good idea?
  - Technical:
    - Are there bugs?
    - Is the code efficient?
    - Does the code implement the desired behaviour?
  - You do not have (1) time or (2) knowledge to do these things for Definitely Typed.
  - Don't even try.

## Implementation

- Social
  - Social knowledge comes from the community, not your personal experience with the PR author.
  - Treat popular packages differently from unpopular ones.
    - For unpopular packages, the primary audience is the package author.
        We want to make sure that the author has a good impression of Definitely Typed and Typescript.
    - For popular packages, the primary audience is new users of Typescript.
        We want to make sure that new Typescript users have a good experience with popular packages.
    - For an unpopular package *none* of the normal code review questions matter.
        The only question is whether other people who care about the package like the change.
        The project board is set up so that you only see packages that already answered that question.
    - For a popular package, we care a lot more.
      - Delegate to the community for the package.
      - Do they trust the author? The author's technical ability?
      - Do they think the change implements the desired behaviour?
      - In certain cases, a change may be too technically complex for owners to evaluate.
      - yargs is a good example of this.
      - In that case, you should give your narrow technical opinon -- "this is cool, and the types seem correct/not too slow"
      - Then explicitly delegate the rest to the owners -- "does this look like a correct change based on the source package?".
  - *Other*
    - Treat unpopular packages whose owner doesn't review just like Already-Approved ones.
    - Popular packages with no reviews should not change.
    - Ask the PR author to contact the owners themselves, perhaps by pinging them on github.
    - You can make exceptions for extremely small changes to packages that aren't react or node.
- Technical
  - The technical part of the review is entirely structural, and applies mostly to new reviews.
  - We only check whether the PR has
    1. An allowed module structure.
       - https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html
       - Briefly:
         1. Global -- no exports, no need for imports.
         2. Commonjs -- one export, with `export = value`, imported with `import x = require('x')`
         3. ES2015 -- multiple exports, imported with `import { a, b, c } from 'x'`
         4. UMD -- (2), but with `export as namespace x` too. Can be used with or without import.
         5. ES2015 default -- `export default value`, almost always wrong.
    2. No overrides in tslint.json.
    3. Correct strictness settings in tsconfig.json:
       - noImplicitAny, noImplicitThis, strictNullChecks, strictFunctionTypes should all be true.
       - Or just "strict": true (allowed since January 2019)
  - Everything else is checked by CI, unless tslint.json has overrides to turn off certain rules.


## Tips

- Read the instructions we give authors:
  - Editing an existing package (with common mistakes): https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/README.md#edit-an-existing-package
  - Module red flags: https://www.typescriptlang.org/docs/handbook/modules.html#red-flags
- Use unpkg to look at shipped code.
  - https://unpkg.com/package-name will give you the latest version's main source file
  - https://unpkg.com/package-name@9.9.99 will give you a particular version
  - https://unpkg.com/package-name/bin/index.js will give you a particular file
  - https://unpkg.com/package-name/bin/ will give you a directory listing
- Use npmjs to determine popularity.
  - Look on the right side for the "weekly downloads number"
- Use project filters to do oldest first.
  - In the "Filter cards" box, type "338" to see PRs that start with 338
  - Then move on to 339, 340, etc.
