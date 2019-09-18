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

How software engineers use data (to make decisions)

1. Effort Estimation
2. New hire ramp up
3. open-ended github mining

NB Argument selection (order?) bug detection

One of the the authors for the Java paper is here, should ask her.
Super-simple name-based matching, handles enums. Google has deployed
it as part of Tricorder

Jaspan, CACM and something else