---
layout: post
title:  "Paper Review - 6"
---

The papers for this week:

1. Bolz et al, Tracing the Meta-Level: PyPy's Tracing JIT Compiler,
   ICOOOLPS'09.
2. Wurthinger et al, Self-Optimizing AST Interpreters, DLS'12.


1.Meta-Level Tracing
--------------------

We have seen multiple approaches of embedding a DSL in different kinds
of host languages:
1. Via a meta-circular interpreter in an interpreted language (eg:
   Lisp, Gedanken), 
2. Via a compiler in a statically compiled language (eg: Elliot's
   Pan), and
3. Via a multi-stage interpreter in a statically compiled language.
   Since a multi-stage program is effectively a code generator, the
   multi-stage interpreter effectively acts as a compiler (eg: DSL in
   Taha et al's A Gentle Introduction to Multi-Stage Programming).
This paper presents an approach to embed a DSL in dynamic languages,
such as Python, which are primarily byte-code interpreted languages.
However, in order to improve performance and stay competitive to
static languages, the run-times of such languages perform dynamic
"just-in-time" (JIT) compilation of hot-paths to machine code. Hot
paths are typically made of loops which execute same block of code
number of times until the guard becomes false and execution branches.
A tracing JIT compiler observes (traces) the loop for multiple
iterations, identifies a sequence of instructions that are frequently
executed, and compiles them to machine code. In subsequent iterations
of the loop, machine code is executed instead of byte-code, yeilding
better performance.

Unfortunately, optimizing hot-paths via tracing JIT compilation does
not extend straightforwardly to DSLs embedded in dynamic languages via
interpreters. The problem is that, while the tracing JIT compiler is
aware of loops in the DSL interpreter, it has no information about
loops in programs written in DSL itself. Since code written in DSL is
effectively the input data to the interpreter written in Python, the
loop structure of the DSL program is an application-level semantic
information, which is not readily accessible to the JIT compiler. The
paper proposes a solution to this problem based on the insight that
DSL interpreters written in dynamic languages often track current
program counter (PC) and current byte-code's opcode using host
language (i.e.) Python variables. The variable being used as PC can be
known by asking "hints" from the programmer. By tracking the forward
and backward movements of the PC variable, it is possible to track the
control in the DSL program. The tracing JIT compiler that uses this
approach has been implemented in the PyPy framework, along with
certain optimizations on the generated machine code, such as constant
folding. The resultant tracing JIT interpreter has been found to
execute DSL programs nearly 3 times faster than an interpreter without
tracing JIT compilation.

An alternative approach to make use of host language's tracing JIT
facilities is to compile DSL programs to host language itself. Since
the host language is anyway restricted to a statically trackable
subset called RPython, such compilation may infact be possible. The
paper makes no reference to this alternative. This could be an avenue
for research.

2.Self-Optimizing AST Interpreters
----------------------------------

Embedded language programs are commonly represented as ASTs in host
language. An interpreter written in host language typically traverses
the AST of the embedded language program "executing" nodes as it
encounters them. As per the paper, straightforward manifestation of this
pattern in Java (using an abstract "ASTNode" class with an "execute"
method) is expensive as virtual method dispatch is costly in Java.
Consequently, language implementers have to adapt alternative
approaches based on bytecodes, which yeild performance at the expense
of simplicity. To remedy this, the paper proposes "Self-Optimizing AST
Interpreters" that reduce the overhead of dynamic method dispatch by
replacing generic AST nodes with type-specialized AST nodes at
run-time. For example an AST node representing a '+' operation in
Javascript can be replaced, based on runt-time profiling information,
to either integer addition operation, or string concatenation
operation, or some application-specific operation. The paper presents
evaluation of the proposed tree rewriting scheme, along with some
optimizations, such as method inlining. The numbers show that
performance improvement is 30-50%.

The proposed method of eliminating virtual method dispatch by type
specialization at run-time seems general enough to be applicable to
Java programs at large. It is therefore intriguing why the paper
proposes this in the context of AST interpreters.


