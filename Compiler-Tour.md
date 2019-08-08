## Parts of a compiler

- Parsing, Binder, Checker, Emitter
- type Parser = (program: string) => Tree<Node>
- type Node = { properties: any[], children: Node[] }
- type Binder = (ast: Tree<Node>) => Map<Node, Symbol>
- type Symbol = { declarations: Node[], members: Symbol[] }
- type Checker = (ast: Tree<Node>, env: Map<Node, Symbol>) => Map<Node, Type>
- type Type = { properties: any, symbol: Symbol }

- type Emitter = (ast: Tree<Node>) => string

## Structure of the talk

- Architecture
- Examples
- Debugging

## Parser (short-ish)
- Architecture: Recursive descent
  - not LR(1), LR(k), LL(1), packrat, PEG, nothing.
  - nothing special here.
  - no guarantees, but in practise things work ok. sometimes we have bugs!
- Examples: probably parseFunction and parseType (and descendants)
- Third example of permissiveness followed up by error checking
- Debug it a bit (need example)

## Binder (also short)
- Architecture: It's a little strange.
  - track a container, which is a node
  - add children to a container
  - for duplicate tracking, use flags to specify legal combinations
  - the result is a 
- Example: bindVariableDeclaration and bindFunctionDeclaration
- Debug it a bit (need example)
  - this will show off the actual code in declareSymbol and friends

## Checker
- Architecture
  - lazily
  - check a given node, which checks dependent nodes
  - can run in get-only mode, opp check mode.
    (in theory -- in 1.1 this didn't really exist)
  - assignability
  - call resolution
  - inference
  - other?
- Examples
  - One for each of the above sections
- Debug it a bit
  - checking a function
  - assignability
  - maybe inference

## Emitter

- Architecture
  - Only uses AST as input
  - This one is tough because it changed so drastically in 2.0.
  - And it doesn't do that much
  - 1.1's code is probably easier to read because of this though!
