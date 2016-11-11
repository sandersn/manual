# Checker reduction plan

1. The checker should not be thousands of lines long since it makes it
harder to learn and contribute to the project.
2. The checker should *at least* be less than 10,000 lines long so that Github will display it.
3. The whole compiler should use modules instead of namespaces.
Breaking the checker into pieces is a good intermediate step.

## Overview

1. Decide on chunks.
2. For each chunk, decide on an implementation.
3. Split the checker into chunks.

The better implementation is a namespace containing functions, a few
of which are exported. References to functions in other chunks can be
obtained by object destructuring of namespaces, either from the main
checker namespace or from other chunks.

But this only works for cohesive chunks that (1) have a few
entry points and (2) don't reference checker-wide state. There are not
many chunks like this.

The more common implementation is a function that contains a set of
functions. The container function takes as arguments the checker-wide
state that the functions in the chunk need to access. It returns an
object containing the 'exported' functions.

In both cases, less cohesive chunks result in long lists of imported
functions and other objects.

## Chunk list

Here's a made-up list of possible chunks.

1. Module resolution: also exports, aliases, etc.
2. Type writing: writeType and getSymbolDisplayBuilder and friends.
3. Type creation?: eg createType, createAnonymousType, etc.
4. Get type from Node: eg getTypeForBindingElement
5. Get type from Symbol: eg getTypeOfAccessors
6. Get base type from type: getBaseTypes and friends.
7. Get type from type?: others?
8. Signature handling: creating, matching, etc.
9. Union and intersections types? Other structured types?
10. Get or resolve information from a [structured] type.
11. Literal types.
12. This types?
13. Evolving (auto) types.
11. Type instantiation.
11. Actual check* functions.
12. Grammar checks.
13. JSX-specific code.
14. JSDoc-specific code.
14. Assignability and other type relations.
15. Type [parameter] inference.
16. Contextual typing.
17. Various type predicates: eg isArrayType, isLiteralType.
18. Control flow.
19. Widening.
20. Call and overload resolution.

