---
layout: post
title:  "Paper Review"
---

The papers are following:
1. Futamura Y, Partial Evaluation of a Computation Process - An
   Approach to a Compiler-Compiler, HOSC 1971
2. Shali A, and Cook W R, Hybrid Partial Evaluation, 2011.

1.Futamura Paper
----------------

The main contribution of the paper is to demonstrate that partial
evaluation is the bridge between definitional interpreter and compiler
of a language. Building on this correspondence, the paper proposes a a
compiler-compiler algorithm that translates a definitional interpreter
to a compiler.

The paper first formalizes the notion of partial evaluation. For a
computation process (call it PI) containing "m+n" variables c1,c2,..,cm, 
r1,r2,...,rn, partial evaluation of PI with respect to c1,c2,..,cn
entails evaluating parts of PI, which can be evaluated by assigning
values c1',c2',..,cm' to c1,c2,..,cm respectively, while leaving rest
of the portions of PI intact. If we denote the partial evaluator as A,
then following equivalence holds:

    PI (c1',c2',...,cm',r1',r2',...,rn') = A (PI,c1',c2',...,cm')
                                             (r1',r2',...,rn')  --> (1)

A definitional interpreter (call it int) of a language (call it Ls) is
also a computational process as described above. If we denote the the
source program to the interpreter as s', and inputs to the source
program as r', then partial evaluation of the interpreter (int) needs
to satisfy the following equivalence:

    int (s',r') = A(int,s')(r')   --> (2)

Let us denote the language in which the interpreter (int) is
implemented as Lint. Since partial evaluator (A) generates residual
code in the same language as its input program, A(int,s') is a program
in Lint that has the same behaviour as the program s' on inputs r'. In
other words, A(int,s') is s' compiled to a program in Lint.

Now, lets consider A(int,s'). Let us dentoe the language of A as La.
Since A is also a computation process, from equation (1), we have:

   A(int,s') = A(A,int)(s')  --> (3)

The above equation means that if we partially evaluate a partial
evaluator (A) with respect to the definitional interpreter of Ls, the
result is a compiler for Ls implemented in La. Since LHS indicates
that A is a partial evaluator for Lint, and RHS indicates that A is a
partial evaluator for La, we require La = Lint. Therefore, equation
(3) conveys a profound fact that if we have a self-applicable partial
evaluator (A) written in a language La, then definitional interpreter
for a new language (Ls) written in La is enough to generate a compiler
that compiles programs in Ls to La. In other words, A is a
compiler-compiler.

How does one construct such a partial evaluator? The paper describes
an algorithm (A1) for partially evaluating a program with loops. The
main feature of the algorithm is how it handles control-flow merge
points. At control-flow merge points, the algorithm checks if values
of static variables is same along multiple merging paths. If yes, then
merge operation over static var-env is trivial, and partial evaluation
continues with merged env. Otherwise, partial evaluation continues
along multiple paths, each with a different static var-env.

The above strategy means that partial evaluation may fail to terminate
(i.e., it produces a residual program of infinite length) even when
the original program terminates at run-time. For example, consider the
following program:

    fun foo k x = if (x>10) then (k,x) else foo (k+1) (x+1)
    foo 0 x

In the above program, for function foo, the binding for k is known
statically. However, the binding differs for different (recursive)
calls to foo. Consequently, partial evaluation keeps unrolling
recursive calls without ever terminating.

The paper acknowledges this problem and proposes a solution. However,
the solution is sketched as a vague text, and is not clear at this
point of time.

2.Hybrid Partial Evaluation Paper
---------------------------------

The paper describes a partial evaluator (PE) for Java programs that
puts pragmatic concerns, namely predictability and reliability of
optimizations, before the effectiveness and efficiency of PE. The two
main aspects of the partial evaluator described in the paper are:

1. Partial evaluation is programmer initiated: Programmer marks
   certain code as compile time, and that code should not refer to
   symbolic inputs. Partial evaluation then makes sure that variables
   with concrete bindings are elaborated away (i.e., no longer appear)
   in the residual program. If such elaboration is not possible (for
   example, see sec 3.7 of the paper), partial evaluation fails.
2. Function (or method) calls with some or all of the arguments as
   symbols are not unrolled. Instead, the function definition is
   specialized for that call-site by substituting concrete values for
   formal arguments whose bindings are concrete at the call-site, and
   a call to the specialized function is inserted at the call-site. If
   two calls to a function have similar context (i.e., same bindings
   to formal arguments and free variables), then same specialized
   function is used at both the call-sites. This approach solves the
   problem of non-termination that plagues other online approaches to
   partial evaluation.

Note about online vs offline PE
-------------------------------

The partial evaluation (PE) process described by the Futamura paper
has come to be known as online partial evaluation. It is online
because the decision of whether to specialize a piece of code with
respect to known constants is made "on-the-fly" during the partial
evaluation time. An unpleasant consequence of online strategy is
potential non-termination, as described in the example above.
Alternative to online PE is offline PE, where a binding time analysis
(BTA) determines what parts of the code can be specialized (and how)
before the partial evaluation process. BTA is a conservative analysis,
which means that it marks only those parts of the code as static on
which PE is guaranteed to terminate. 




