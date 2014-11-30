---
layout: post
title:  "Paper Review - 2"
---

The papers for this week:

1. Elliot's paper: C Elliot, S Finne, and O de Moor, Compiling
   Embedded Languages, JFP, 2003.
2. The 700 paper: P J Landin, The Next 700 Programming Languages,
   1965.

1.Elliot's paper
-----------------

In the Lisp paper and Reynolds's Definitional Interpreters paper, we
have seen how a language can be _defined_ by writing an interpreter in
a different ("hopefully better understood") language. In such cases,
any _code_ (eg: M-expressions) in the defined language becomes _data_
(eg: S-expressions) for the interpreter written in defining language;
when we execute the interpreter, we effectively execute the code in
defined language. An example is an interpreter for STLC written in
SML; we define STLC _code_ as an algebraic data type in ML, and when
we execute the interpreter, we execute STLC code.

Consider an alternative approach: instead of embedding a language
by writing an interpreter, we embed it by writing a compiler. A's Code is
again B's data, but executing the compiler in B compiles A's code
instead of executing it. If compilation produces a program in a
different language C, and if semantics of C are well-understood, then
the compiler written in B can be considered a _definitional compiler_.

However, most often, the reason for embedding A within B via a compiler
is not just for the sake of a definition. Along with the usual
benefits of embedding, it gives us 1. performance, and 2. Ease of
writing a compiler.

In this paper, Elliot et al describe a technique for easily writing
optimizing compilers for DSELs. They illustrate their technique with
via a DSEL call pan for for computationally intensive domain of image
synthesis and manipulation. The original idea was proposed by Kamin,
but he used strings to represent program fragments (Shallow
embedding). Elliot uses algebraic datatypes (Deep embedding), and
shows that it greatly facilitates compile time optimizaton.

For instance, he defines FloatE, BoolE and IntE to represent float,
boolean and int expressions respectively. For functions and tuples
though, he uses Haskell's (i.e., hybrid approach). But representing a
fn as haskell fn means we cannot observe its body. To get around this
restriction, he adds a VarFloat, VarBool and VarInt constructors to
those types. To observe the body of the fn, he then applies fn to
these Var types.

DSELs let us do compile-time computation. For instance, a rotate
function written in over haskell values performs computation at
run-time, whereas a rotate fn written over DSEL representation of
points runs and produces an optimized code that may not even have to
call the rotate function when it runs.

Sometimes, it may be necessary to layer compilation of DSEL into
number of distinct abstract layers. A compiler x in B can run and
compile the program p in A to q in A. This can again be compiled by
compiler y in B to program r in A, and this will go on.


In 





