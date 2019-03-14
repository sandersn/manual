# Introduction

Typescript began its life as an attempt to bring traditional object-oriented types
to Javascript so that the programmers at Microsoft could bring
traditional object-oriented programs to the web. As it has developed, Typescript's type
system has evolved to model code written by native Javascripters. The
resulting system is powerful, interesting and messy.

# Things you should know

In this introduction, I assume you know the following:

- Syntax of a C-descended language, including the syntax for type parameters.
- Semantics of a mainstream object-oriented type system.
- How to program in simply typed or untyped lambda calculus or
  Javascript, "the good parts"

I do not recommend writing object-oriented programs in Javascript or
Typescript, but the type system unavoidably relies on concepts common
in OO in a few places.

# Things that might be new

- Javascript primitive types
- structural typing
- gradual typing
- merging
- unit types

# Things that are like Haskell/ML

But different. Usually worse. 

- contextual typing
- untagged unions
- discriminated unions
- type aliases
- tagged [unit] types
- ad-hoc replacements for The Popular Monads
- modules

# Advanced types that are Javascript's fault

- Mapped types

You know what? This section is so tiring. Go read about these types on their own pages.

# Advanced types that are not in Typescript

- Pattern matching
- Special syntax for the Maybe monad
- Special syntax for the List monad
- Lists at all
- Higher-kinded types

So don't try to write any monads!
