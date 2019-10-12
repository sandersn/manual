When I'm working on Typescript's infer-from-usage codefix, which looks
at usage of variable to infer their types, I need to make sure that
I'm making progress on an ill-defined problem with many aspects.
Because of the ad-hoc nature of the inference, it's easy to improve
one aspect while making another worse. I need to know whether I'm
improving the system as a whole.

## Overview

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
   don't have a reasonble type, like the keywords `import` or `if`, for
   example.
4. Compare before/after: the number of compile errors.

The counts of `any` and errors allow me to roughly track performance
over time. And since the script leaves the second clone of Typescript
in the post-codefix state, I can inspect the user tests to learn how
the codefix works on actual code. The combination of overall
performance combined with ability to inspect individual changes lets
me make progress on a problem with a defined end-goal.

## Implementation

The scripts do some git and gulp manipulation which isn't that
interesting to build the new Typescript comment. Then they create a
language service. Notably, I use a standard interface (usually
typescript@next) and cast the newly-compiled language service to that interface.