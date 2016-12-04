Once upon a time in grad school I got a
[real live Ph D](https://github.com/sandersn/dialect) in computational
linguistics, and programming languages were just my hobby. Back then I
bored my colleagues talking about lambdas all day. Now writing a
compiler is my job and linguistics is my hobby. So now it's the
Sapir-Whorf hypothesis that drives my colleagues to sneak away from the
lunch table while I keep talking.

Anyway, I forked the
[`natural`](https://github.com/NaturalNode/natural) package recently
to check out the state of natural language processing in the
Javascript world. I decided to upgrade it to Typescript so I would
know that I really understood the code. So far it's been more of an
exercise in upgrading a package to Typescript, so I thought you might
want to follow along to see what's involved. Here are the topics I
plan to cover:

1. Compile using Typescript
2. Switch to gulp
3. Actually upgrade to Typescript
4. Acquire types to make further development easier
5. Polish and make things stricter

# Step 1: Compiling with Typescript



# Addendum: Full outline

1. Add tsconfig
  a. Fix bare-minimum compile errors
2. Switch to gulp
  a. Copy things over to src/ from lib/
  b. Add tasks to compile, then copy
  c. Add tasks to copy other stuff
  d. Add task to test
3. Actually upgrade to TS
  a. Add types
  b. Add a types file
  c. Upgrade to ESNext features
4. Acquire types
5. Polish
  a. Upgrade to noImplicitAny
  b. Upgrade to strictNullChecks
Publish your own types
