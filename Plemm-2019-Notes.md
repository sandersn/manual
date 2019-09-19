# Wednesday first

## Adverserial Examples for Code Models

1. AST paths are surprisingly powerful for
building models because they encode so much info in programming
languages.

They use two-ended paths, which sound way better (and more expensive)
than paths-to-root.

2. AI models for predicting identifiers, new code (tree or next
token), types, etc are subject to subtle AI-based attacks. Important
when your npm vuln scanner uses AI.

3. Quick explanation and lots of examples of how to build attacks
(they tried *defending* but couldn't get that part to work).

## Program Representatinos for Deep Learning
  Mark Brockschmidt (from MSR)

Lots of discussion of why these models have trouble compared to
natural language. FLip side of previous talk.

Training data is hugely lacking for many tasks.
Models for English are so-so.
Reference patterns are very non-natural.
Order is more? less? important than English or (say) Russian.

Different models

- Linear
- "Attentional" (linear dependency)
- Tree
- Tree paths
- Graph w/dataflow


Work around data sparsity ... how?
Work around computation cost ... how?
Work around interproc analysis ... how? (Facebook's infer tries to
compute summaries)
Work around language dependence ... how? (LLVM-based IR...uh?)
Work around research reproducibility ... how? (NLP is pretty well-off
here, comparatively speaking)

He's not giving any good answers still. Just naming off some names
that aren't great

Uh, and here's his summary: It doesn't work very well yet.

## Data and software engineering

(Another MS speaker)

How software engineers use data (to make decisions)

1. Effort Estimation

He uses this to nudge people to complete PRs (not totally sure how
this is connected)

2. New hire ramp up

3. Geographical bias in github PR selection

compared to US submitters
UK, Ca, JP, NL, Switzherland had a HIGHER success rate than US

Germany, ... had LOWER

Same nationality or region, goes up by 19%

Perceived and real bias were neither one affected by language.

Perceived bias only existed against India, both from submitters and integrators.
But real bias didn't exist for India.

4. End to end machine learning development

## ML+IR for Scalable Crash Resolution

(Information retrieval)

This is not relevant to TS so I didn't listen.

## improving Software Reliability using ML

Large scale data analysis in github said 88% were trivial errors.
This talk about more ways to do that.
Discussion of fuzzing, problems and definition.
Basically, they used ML to reduce the search space of fuzzing.
The noteworthy part of the research is how well it performs. It's
really good.


## Improving Engineering Productivity at Scale

Google engineer from Engineer Productivity Resarch

How correct do codefixes have to be?

This is an explanation of the scientific method as applied to
business. Not too technical.

## Neural Program Testing

Fuzzing for Security Bugs
Yep it's another fuzzing talk.
Grammar-based fuzzing
Writing a grammar is annoying and hard
Inferring a grammar is annoying and lossy

OK so this is just learning a grammar.

Latent representations of grammar. ok ok.

Also an agent that walks around a state space to fuzz the view.

Also repairing student programs. I have no idea how this all relates.

## Github semantic language support

Github's library for multi-lingual static analysis
- code navigation
- abstract interpretation

Semantic is written in HaSKeLl. Long explanation follows. But they're
writing a compiler.
Blob -> parse using tree-sitter
convert to generalised asts
then...Analyse. That's it!

why tree-sitter?

Most of the talk is about writing the "convert to generalised asts"
step, which used to be a tedious conversion step from the C library.

Oh no it's super specific to github's implementation yet reminiscent
of problems I've had in the past. I'm learning nothing yet having
flashbacks.

Template Haskell -- better than .TT files in .NET????

## From conversation

NB Argument selection (order?) bug detection

Ciera Jaspan, super-simple name-based matching, handles enums. Google has deployed
it as part of Tricorder
https://dl.acm.org/citation.cfm?id=2818828

Jaspan, CACM and something else