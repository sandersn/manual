# Checker reduction plan

1. The checker should not be thousands of lines long since it makes it
harder to learn and contribute to the project.
2. The checker should *at least* be less than 10,000 lines long so that Github will display it.
3. The whole compiler should use modules instead of namespaces.
Breaking the checker into pieces is a good intermediate step.

## Overview

1. Decide on chunks.
2. For each chunk, decide on an implementation.

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

### Survey

This is a boring in-order survey of what's in the checker. I'll use
this list to see what can be moved out to a separate file.

1. `nextSymbolId, nextNodeId, nextMergeId, nextFlowId` and associated
   `get` functions. These are core to the state of the checker, so
   should not be moved. (Or moved very carefully.)
2. `createTypeChecker`. This creates a checker. Don't move this!
3. getEmitResolver and error. Miscellaneous?
4. createSymbol to addToSymbolTable. Manage Symbols and SymbolTables.
5. getSymbolLinks, getNodeLinks. Manage the look-aside tables the
   checker uses to avoid modifying binder data structures.
6. isGlobalSourceFile. ???
7. getSymbol, getSymbolsOfParameterPropertyDeclaration. More SymbolTable -- look up in SymbolTable, but with
   resolving aliases.
8. isBlockScopedNameDeclaredBeforeUse. What it says on the tin. Weird
   placement tho.
9. resolveName. Resolve a Node to a Symbol. ...and that's 5% done!
10. checkAndReportErrorForMissingPrefix. Nice error for forgotten
   'this'.
11. checkResolvedBlockScopedVariable. error based on isBlockScopedNameDeclaredBeforeUse.
12. isSameScopeDescendentOf, getAnyImportSyntax, getDeclarationOfAliasSymbol. Misc???
13. getTargetOfImportEqualsDeclaration to getTargetOfNamespaceImport.
   Resolve import Nodes to Symbols. Relies mostly on
   resolveExternalModuleName, which is a bit lower.
14. combineValueAndTypeSymbols. Merge symbols.
15. getExportOfModule, getPropertyOfVariable. These don't belong --
   they are just complex specific helper code on Symbol.
16. getExternalModuleMember to getTargetOfAliasDeclaration. Goes with
   other getTarget* functions.
17. resolveSymbol, resolveAlias. resolve aliases (from modules?) to aliases?
18. markExportAsReferenced, markAliasSymbolAsReferenced. Out of place.
   Used way lower, when checking imports.
19. getSymbolOfPartOfRightHandSideOfImportEquals to (at least)
   resolveESModuleSymbol. The actual code behind getTarget* code.
20. hasExportAssignmentSymbol to getExportsForModule. Find exports of various symbols, particularly modules.
21. getMergedSymbol to symbolOfValue. Utilities on Symbol.
22. findConstructorDeclaration. ooplace, but what it says on the tin.
23. createType to createObjectType. Create types.
24. isReservedMemberName, getNamedMembers. An interleaving of utilities to do with symbols again.
25. setObjectTypeMembers, createAnonymousType. Create/update types.
26. forEachSymbolTableInScope. Another Symbol/SymbolTable utility.
27. getQualifiedLeftMeaning to isEntityNameVisible. Figure out symbols in modules.
28. writeKeyword to getTypeAliasForTypeLiteral, plus getSymbolDisplayBuilder. Display symbols. Lots of code here.
29. isTopLevelInExternalModuleAugmentation. Sort of detector for module augmentations. Purely
syntactic and definitely out of place -- called thousands of lines later.
30. isDeclarationVisible, collectLinkedAliases plus getDeclarationContainer. More module information.
31. pushTypeResolution to popTypeResolution. Core type resolution machinery.
32. getDeclarationContainer. Out of place, see above.
33. getTypeOfPrototypeProperty to at least getTargetType. Type creators and associated utilities.

