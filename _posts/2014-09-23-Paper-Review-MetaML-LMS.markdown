---
layout: post
title:  "Paper Review - 3"
---

The papers for this week:

1. MetaML, TCS'00
2. LMS

1.MetaML
--------

This paper introduces MetaML - the first high-level programming
language for multi-stage programming. Multi-stage programming is a
kind of generative programming where each stage generates code to be
executed in the next stage. This paradigm of programming is very
useful in constructing high-level abstractions and parameterized
generic programs which can be elaborated to non-parametric low-level
code before execution, effectively imposing zero or very low
performance cost in return for using abstractions. Another good
usecase is embedding domain-specific languages (DSLs). There are
primarily two ways of embedding a DSL within a general-purpose
language - via an interpreter (ala Lisp) or via a compiler (eg:
Elliot's Pan). While former method is easier to implement, the later
method leads to more effecient code. Embedding a DSL in a multi-staged
programming langauge lets us combine advantages from both the
approaches - A DSL can be implemented via a multi-staged interpreter
program, which can be elaborated in multiple stages to effecient
low-level code.

At its core, a MetaML program is simply an ML program with added
annotations (brackets, escapses and "Run"s) specifying the evaluation
order of various computations of the program. Two main distinguishing
aspects of MetaML are:
1. Hygiene: MetaML allows variables from later stages to be referred
   to in earlier stages. It allows cross-stage persistence (CSP) while
   retaining static scoping and avoiding name captures.
2. Static Typing: A well typed MetaML program is guaranteed to
   generate well typed ML code. There is no need to type check
   the generated code again. 

There is one major disadvantage with MetaML approach of multi-stage
programming - one cannot observe the structure of the program, even
the parts that belong to earlier stages. The results of previous stage
are conveyed to next stage in form of values of type ['a code], where
['a] is the actual type of the value generated. All that the next
stage can do now is to inline the ['a code] or just "Run" it. The
inability to observe the structure of the program means optimizations,
such as algebraic optimizations cannot be performed.

2. LMS
-------

Lightweight Modular Staging (LMS) is an approach to multi-stage programming
that is fundamentally different from that of MetaML. As the name
indicates, it is lightweight, which means two things:

1. At a superficial level, it does not need addition of language
   constructs like quasi-quotation to enable staging. .
2. More importantly, it does not require staging to be baked into
   operational semantics of the language. Instead, it can be
   implemented as a library atop "a sufficiently expressive" langage.
   A staged program can still be understood using the conventional
   semantics of the host language.

LMS makes use of type annotations to distinguish between stages: a
single staged expression of type T has type Rep[T]. Modularity comes
from the fact that for each type T, its staged expression (Rep[T]) can
be defined by a module (component in case of Scala) which defines
concrete data types to represent the expression, implements required
operations, along with possible optimizations, such as constant
folding. However, thanks to data abstraction, this internal
representation is concealed from the staged code, preventing it from
examining itself, thereby restoring safety. Hence LMS approach allows
compiler optimizations while retaining safety.

One disadvantage with LMS approach when compared to MetaML approach is
that while the later allows any language construct to be staged at
any level, LMS requires defining staging components explicity.
