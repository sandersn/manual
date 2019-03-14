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
    - Treat unpopular packages whose owner doesn't review just like Already-approved ones.
    - Popular packages with no reviews should not change.
    - Ask the PR author to contact the owners themselves, perhaps by pinging them on github.
    - You can make exceptions for extremely small changes to packages that don't seem THAT popular.
    - For example, they're popular, but have a really dumb name.
- Technical
  - The technical part of the review is almost entirely structural.
  - We only check whether the PR used an allowed module structure.

## Tips

- Use unpkg to look at shipped code.
- Use npmjs to determine popularity.
- Use project filters to do oldest first.
