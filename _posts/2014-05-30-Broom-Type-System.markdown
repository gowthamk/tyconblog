---
layout: post
title:  "Region Type System - Motivation and Related Work"
---

Introduction
============

In this document, we motivate the need for a region type system for
Broom programs, clearly identify the problems that the type system
needs to solve, and discuss some related work along with the
applicability of proposed solutions to our problem setting.

Please note that since this is a publicly available document, I do not
provide any code examples. Nevertheless, the reader is highly
encouraged to go through the examples provided in OneNote document
before reading this writeup.

System Description
==================

For the sake of the discussion, we assume that system comprises of a
set of actors, each with an Actor Id (_actId_). An actor is an
instantiation of a class that belongs to an application. All actors
run on the same machine sharing same physical memory, and communicate
by message passing. 

Memory Management
=================

In Broom, memory is allocated to actors in terms of blocks called
regions. Three kinds of regions can be identified in Broom:

+ *Private Regions* : A private region is a block of memory that is
  private to the actor to which is was allocated. Memory for local
  variables of methods within the actor, and data structures that are
  private to the actor are allocated in its private regions. Actor's
  private regions can be made to abide by the LIFO discipline, with
  following characteristics:

  1. A _private stack region_ is allocated to an actor whenever it
     executes a method call. Such region essentially serves as an
     activation record for the method being called. As with activation
     records, private stack regions form a stack of regions, where
     regions at higher addresses in the stack (stack is assumed to
     grow downwards) have longer lifetime than those at lower
     addresses. A private stack region can be deallocated as soon as
     its corresponding method call returns.
  2. A _private heap region_ is allocated to an actor at the beginning
     of actor's life time, and outlives all its stack regions. Data
     stored in actor's heap region is assumed either to persist
     throughout the lifetime of the actor, or to be garbage collected
     appropriately. In other words, there is no explicit way to
     deallocate memory in private heap region.

+ *Transferable Regions* : A transferable region is a block of memory
  that is allocated/deallocated to an actor on demand. Subsequent to
  the allocation, the actor to which the transferable region is
  allocated is its unique _owner_. Owner of a transferable region can
  allocated/deallocate objects into the region. An actor can chose to
  transfer the ownership to another actor (whence _transferable_
  region). Subsequent to the transfer, the former is no longer allowed
  to refer into the region. It can be observed that a transferable
  region and the act of ownership transfer effectively model a
  message, and the act of passing the message between different
  actors, respectively.  Ownership transfer effectively ends the
  lifetime of transferable region as far as its owner is concerned.
  Alternatively, the owner of a transferable region can chose to
  deallocate it, marking the real end of region's life time.
  Transferable regions mark the departure from LIFO discipline among
  lifetimes of regions. Some important observations regarding this
  point:

  1. Depending on when it is allocated and deallocated, a transferable
     region can outlive multiple private (stack) regions of its owner,
     or it may outlive none. Example for first case is when a
     transferable region is allocated at the beginning of actor's
     lifetime, updated throughout its lifetime via multiple method
     calls, and is transferred to another actor just before the end of
     owner's life time. An example for second case is when the
     ownership of the transferable region is transferred to a
     different actor almost immediately after it has been allocated
     (i.e., allocation and ownership transfer are not separated by a
     method call). Nevertheless, LIFO discipline is not violated in
     both the examples, as the lifetime of regions can still be
     represented as in form of a stack involving private regions and
     transferable regions. However, LIFO discipline is violated when a
     transferable region allocated in a method A, is deallocated in a
     method B called subsequently (i.e., activation record for B
     occurs below that of A). One consequence of this is that a
     local variable of A that points into the transferable region
     suddenly becomes a dangling pointer after the call to B.
     Evidently, lifetimes of regions cannot be represented as a stack
     in such case. 
  2. An actor can own multiple transferable regions. Lifetimes of such
     regions can overlap, but need not necessarily be ordered by
     the $${outlives}$$ relation. An example of such case is when
     two transferable regions are deallocated in different order than
     they were allocated. Neither outlives the other in such case.
     Pointers from one region to other become dangling when the later
     is deallocated before the former. A more complex case is when
     ownerships of both the allocated regions are transferred.
     Pointers between transferable region become dangling depending on
     whether ownerships of both are transferred to the same actor, or
     different actors.

  (I presume that) In order to avoid aforementioned complexities,
  we impose certain restrictions on how Broom programs should deal
  with transferable regions:

  1. We require that allocations and deallocations within a transferable
     region happen within a lexical block, during which the region is
     $$open$$. An open region cannot be deallocated; neither can it be
     transferred. Therefore, a call to a method made within the
     lexical block that opens the region cannot deallocate or transfer
     the region. There can be multiple non-overlapping lexical blocks
     which open a given transferable region. Consequently, the entire
     lifetime of a transferable region is split into multiple
     incarnations, where each incarnation is in $$outlives$$ relation
     with lifetimes of actor's private regions. Effectively, we have
     reinstated the LIFO discipline among the lifetimes of private
     regions and lifetimes of transferable regions. With this
     restriction, a dangling pointer manifests straightforwardly in
     the syntax of the program - a pointer into a transferable region
     is dangling if its scope exceeds the scope of lexical block during
     which the region is open.
  2. We require that a transferable region be closed with respect to
     pointer references. That is, a transferable region should not
     contain pointers into other transferable regions, or even the
     private regions of an actor. With this restriction in place, the
     question of formulating an outlives relation among transferable
     regions is now moot. 

+ *Shared Regions* : Shared regions are regions that are always live,
  and shared among multiple actors. We ignore share regions for the
  time being.

Region Types
============

Ensuring memory safety is equivalent to ensuring that

+ There are no space leaks (or region leaks, in presence of explicitly
  allocated regions), and
+ All memory accesses are safe. A dangling pointer (i.e., a pointer
  into a deallocated region) is never followed.

In the absence of explicitly allocated transferable regions, ensuring
memory safety of Broom programs is only as hard as ensuring memory
safety of C programs without `malloc` and `free`(recall that private
heap region of an actor is either immortal, or automatically garbage
collected). Since there are existing solutions that work well for this
subset, we concentrate on containing unsafe interactions with memory
that are possible due to the introduction of transferable regions.
Such interactions can be understood by examining the following state
machine representation of the lifetime of a transferable region:

![TRLifeCycle]({{ site.baseurl }}/assets/broom-tr-lifecycle.jpg)

The above figure succinctly captures all legal interactions with
transferable regions in a Broom program. A transition not represented
in above state machine is illegal. For example, following are some
illegal actions that violate memory safety:

1. Any action on a region that is not yet allocated.
2. Closing a closed region, or opening an open region.
3. Reading from, or writing to, a closed, freed, or transferred
   region. To state more precisely, following a pointer into closed,
   freed, or transferred region is a violation of memory safety.
4. Freeing, or transferring an open region.
5. Any action on the region that is transferred, or freed.

Our goal is to formulate a region type system primarily to prevent
aforementioned bad behaviours. To be more specific, our plan is to:

1. Define a model (a calculus) that can capture all the aforementioned
   unsafe operations, yet sufficiently constrained such that reasoning
   with the model is easy.  
2. Define operational semantics, such that the evaluation gets _stuck_
   if a memory unsafe operation is attempted.
3. Define static semantics for the type-annotated programs of the
   calculus, such that type checking is a) decidable, and b)
   practical. Prove type safety (Well typed programs do not get
   _stuck_).
4. Construct an algorithm for type inference. Type inference may not
   be complete; our aim is to minimize manual annotations as much as
   possible.

Related Work
=============

We now closely examine some of the related work in the area of region
type systems. Specifically, we will describe salient aspects of the
following papers, while commenting on the applicability of the
solution proposed to our problem setting:

+ Tofte & Talpin, Implementation of the typed call-by-value λ-calculus
  using a stack of regions, POPL'94.
+ Grossman et. al., Region-Based Memory Management in Cyclone,
  PLDI'02.
+ Henglein et. al., A direct approach to control-flow sensitive
  region-based memory management, PPDP'01.

Due to the unconventional semantics of transferable regions, our
problem setting is somewhat unique; so none of the aforementioned
works apply directly in our case. Nevertheless, our motivation is to
identify differences, and use their models as starting point to
construct right model for our setting.

Tofte and Talpin
================

### Summary ###

Introduces statically nested memory regions for ML programs.
Elaborates well-typed ML programs, introducing following expressions
($$\rho$$ denotes regions, $$e$$ denotes expressions):

$$
  {\sf letregion} \; \rho \; {\sf in} \; e \\
  e \;{\sf at}\; \rho
$$

We denote ML extended with above expressions as ML+$$\rho$$.  $${\sf
letregion}$$ expression of ML+$$\rho$$ introduces (allocates) a
region, which is bound to $$\rho$$ in $$e$$. The region handler
($$\rho$$) can be used in expressions form $$e' \;{\sf at}\; \rho$$
within $$e$$ to store the result of evaluating $$e'$$ (a runtime value
- an integer, a tuple, or a closure) in $$\rho$$. The lifetime of a region
introduced via $${\sf letregion}$$ expression is apparent from its
lexical structure; when $$\rho$$ goes out of scope, the lifetime of
corresponding region has ended and it can be deallocated. Since all
runtime values are stored in regions, lexical scoping of regions
essentially means that lifetimes of runtime values are statically
determined (or conservatively approximated). A most conservative
elaboration algorithm from ML to ML+$$\rho$$ would simply enclose
entire program inside one $${\sf letregion}$$ expression. This
approach entails:

+ Making a conservative assumption that all runtime values are always
  needed, and
+ Using a single region to store all such values, which is never
  deallocated.

Such an approach, though sound, does not help in memory reuse,
defeating the primary purpose of region-based memory management. A
good elaboration algorithm should localize lifetimes of runtime values
as much as possible. Therefore, it should introduce several $${\sf
letregion}$$ annotations, making regions regions as disctinct and
local as possible. Such an elaboration scheme is the primary
contribution of the paper.

In the absence of closures, determining the lifetime of values is
relatively straightforward - one only has to take the scope of value
in to account. For eg, consider the ML code:

    let x = 2 
    in let y = let z = 1 in x-z
    in x+y

Its elaborated version is given below:
    
    letregion r1 
    in let x = 2 at r1 
       in letregion r2 
          in let y = letregion r3 
                     in let z = 1 at r3 
                        in (op -) (r1,r3,r2) (x,z)
             in (op +) (r1,r2,r4) (x,y)

Regions $$r1$$, $$r2$$, and $$r3$$ hold the values bound to variables
$$x$$, $$y$$, and $$z$$ (resp.), and have same scope as the variables
themselves. For example, the scope of region $$r3$$ is the
(elaborated) expression $$x-z$$, after which it can be deallocated.
The deallocation is safe, as $$z$$ is not needed outside its lexical
scope. 

In the above example, Functions $$+$$ and $$-$$ are assumed to be
region polymorphic, meaning that they are parameterized over regions
of their arguments and also the region where the result has to be
stored. For instance, (op +) is (`+` in the below definition denotes
machine integer addition):

    Λ(ρ0,ρ1,ρ2). λ(a: int@ρ0, b : int@ρ1). a+b at ρ2.

In the paper, the _decorated type_ $$int@ρ$$ is written $$(int,ρ)$$,
and denotes an integer at region $$ρ$$. Instantiating the
parameterized region variables of `(op -)` with $$(r1,r3,r2)$$ yields
a subtraction function that expects its arguments at $$r1$$ and
$$r3$$, and stores the result in $$r2$$. Since this result is bound to
variable $$y$$, whose scope is the (elaborated) expression $$x+y$$,
region $$r2$$ has to be live at $$x+y$$, which is ensured by the
corresponding $${\sf letregion}$$ annotation.

It can be observed that region $$r4$$ occurs free in the elaborated
version. This is because $$r4$$ is the region that holds the result of
evaluating the above expression; so, it has to be live even after the
expression is finished evaluating.

This simple scheme of using the lexical scope of the variable to
determine the lifetime of the bound value fails in presence of
closures. Consider, for example, the following simple ML expression:

    let x = (2,3) in λy.(fst x,y)

The lexical scope of $$x$$ is only the lambda expression under $${\sf
let}$$, but the runtime value bound to $$x$$ escapes the scope of
$$x$$ through the closure. In other words, the lifetime of runtime
value bound to $$x$$ exceeds the lexical scope of $$x$$. In order for
the elaboration to ML+$$\rho$$ to be memory safe, it is imperative to
keep track of values that _hide_ under closures and escape the lexical
scope of their names.

The paper introduces _arrow effects_ and _effect variables_ for this
purpose. An effect in the context of the paper is either a _read_ or a
_write_ to/from a region. The intuition is to label the arrow type
(i.e., type of a closure) with _read(ρ)_ and _write(ρ)_ tags, where ρ
is a region, such that they indicate regions that could be
read/written if the closure is executed. For example, the `fst`
function, which projects the first component of an integer pair, could
have the following type:

    fst : ∀(ρ0,ρ1,ρ2). ((int@ρ0, int@ρ1)@ρ2) -φ-> (int@ρ0)

The label (φ) over the arrow denotes the following effect set:
$$\{read(ρ2),read(ρ0)\}$$. Now, assuming that the expression `(2,3)` is
elaborated to `(2 at r1, 3 at r2) at r3`, for some regions $$r1$$,
$$r2$$, and $$r3$$, the lambda expression:

    λy. (fst x,y)

will surely have the arrow in its type labelled with the effect set:
$$\{read(r3),read(r1)\}$$, which tells us that if the corresponding
closure is evaluated, regions $$r1$$ and $$r3$$ are read. Since the
closure escapes the $${\sf let}$$ expression, $$r1$$ and $$r3$$ should
not be generalized (with $${\sf letregion}$$) in this expression. On
the other hand, $$r2$$ can be generalized:

    letregion r2
    in let x = (2 at r1, 3 at r2) at r3
       in ....
      
Region $$r2$$ can be safely deallocated once the $${\sf let}$$ is
evaluated, as the values stored in $$r2$$ do not escape through the
closure.

The paper formalizes intuitions stated thus far and proposes an
elaboration scheme together with a region type discipline for ML
programs. The formalization accommodates parameterization over regions
and effects in function types, which invariably leads to multiple
categories of type schemes. It is not clear at this point of time how
how type safety was phrased and proved for the region type system.

### Comparision with Broom ###

After discussion with Rama, I realized that private regions are
intended to be statically nested regions similar to Tofte and Talpin
(T&T) regions. They need not necessarily be activation records.
Further, it is (currently) assumed that Broom programs have
annotations introducing static regions; so, at first glance, it
appears that the target language of T&T could be sufficient to model
Broom programs with only private regions. In such case, Region type
system of T&T could be reused for this subset. However, some
questions:

1. In the target language of T&T, $${\sf letregion}$$ syntax ensures
   LIFO discipline among static regions. Is there any such lexical
   block to introduce a static region in Broom programs? Since static
   nesting essentially means lexical nesting, I expect that we should
   be able to define a syntactic block to introduce static regions.
2. Consider a pair of statically nested regions, $$r1$$ and $$r2$$,
   such that $$r1$$ outlives $$r2$$, and an expression $$e$$, which
   can see (i.e., is in scope of) both $$r1$$ and $$r2$$. In T&T
   target language, the example of such an expression is:

        letregion r1
        in ....
           letregion r2
           in e
  
   $$e$$ can store values in $$r1$$ and $$r2$$; That is, $$e$$ can
   contain sub-expressions of form $$e' \; at \; r1$$ and $$e' \; at \;
   r2$$. However, it was mentioned during the discussion today that
   under semantics of regions in Broom, sub-expression of later form is
   allowed, but the former sub-expression is disallowed. Futher, it was
   also mentioned that $$e$$ can _allocate_ memory in $$r2$$, but
   cannot _store_ a value in $$r2$$. This difference is not clear.

Nevertheless, T&T language has no non-statically nested regions, which are
required to model transferable regions of Broom. Further, the language
has no references needed to capture the possibility of dangling
pointers in Broom programs that result from target region being
freed/transferred. 

Cyclone
=======

<!-- A region is a logical container for objects that obey some memory
management discipline. For instance, a stack frame is a region that
holds the values of variables declared in a lexical block, and frame
is deallocated when control flow exits the block. Garbage-collected
heap is another example, whose objects are individually deallocated by
the collector.
-->

Cyclone is a type-safe subset of C. The primary purpose of the type
system is to ensure memory safety while allowing programmers to
have control over memory management. Here are the  main ideas:

### Region Type System ###

consider a subset of C with only stack allocated memory (i.e., only
local variables are allowed). This subset is still memory unsafe, as
it is possible for a method to return the address of a local variable to
its caller, which then becomes a dangling pointer. A simplified
version of such example can be presented using lexical blocks:

{% highlight C%}
    int *p;       //1
    L : {         //2
      int x = 0;  //3
      p = &x;     //4
    }             //5
    *p = 42;      //6
{% endhighlight %}

Since memory for variable $$x$$ is deallocated as soon as lexical
block $$L$$ ends, the statement `p = &x` effectively makes $$p$$ a
dangling pointer at line 6. However, this unsafe operation can be
detected by observing that :

1. Variables $$p$$ and $$x$$ belong to different lexical blocks,
2. Lexical block of $$p$$ outlives that of $$x$$, and
3. $$p$$ refers to $$x$$, which means that $$p$$ points to deallocated
   memory after lifetime of block $$L$$.

T&T's region type system allows us to perform such reasoning. To see
how, consider the following program in T&T's target language (extended
with references), which is equivalent to above C code:

    letregion r1
    in let p = letregion rL
               in let x = 0 at rL
                  in (ref x) at r1
       in p := 42

The assignment on last line is an unsafe operation as $$p$$ is a
reference into region $$rL$$, which is now deallocated. Fortunately,
the program is ill typed under T&T's region type system, as decorated
type of expression `ref x` is $$({\sf ref} \; ({\sf int},rL),\, r1)$$,
which must be the type of $$p$$ as per the type rule of $${\sf let}$$
expression; but, such a type for $$p$$ is ill-formed as region $$rL$$
is not in scope.

Cyclone makes use of the above insight to judge ill-typedness of the C
program shown previously. A lexical block is considered a region in
Cyclone. Static nesting among lexical blocks gives rise to the a stack
of regions at run-time, which is effectively the C call stack. The
region type system of T&T can now be applied to ensure memory safety.

### Dynamic Regions or LIFO Arenas ###

Cyclone introduces dynamic regions with lexically scoped lifetime. The
name "dynamic region" is somewhat a misnomer as dynamic regions of
Cyclone are delimited by a lexical block, just like stack regions.
However, Cyclone allows unlimited allocations into dynamic regions via
`rmalloc` and `rnew` primitives. The allocated memory gets deallocated
in tandem with the dynamic region, when the control exits the lexical
block corresponding to the dynamic region. From the perspective of
region type system, it is not interesting to distinguish between stack
regions and dynamic regions.

### Garbage Collected Heap ###

Memory that needs to survive limits of lexical blocks is dynamically
allocated on heap through the usual `malloc` call. There is no `free`,
hence no dangling pointers. Garbage collection is used to reclaim heap
memory.

### Outlives Relation & Subtyping ###

The LIFO discipline on region lifetimes gives rise to _outlives_
relationship among region lifetimes, which allows Cyclone to define
subtyping relation among pointer types. This subtyping discipline is
quite useful in the context of C. For instance, consider the following
example:

{% highlight C%}
    int p = 0;        //1
    L : {             //2
      int q = 1;      //3
      int *x = (p<q)? //4
          &p : &q;    //5
      return *x;      //6
    }              
{% endhighlight %}

Here is a similar example in T&T target language:

    letregion r1
    in let p = 0 at r1
       in letregion rL
          in let q = 1 at rL 
             in let x = if p<q then (ref p) at rL 
                               else (ref q) at rL
                in !x
    
The types of $${\sf then}$$ and $${\sf else}$$ branches of the $${\sf
if}$$ expression are $$({\sf ref} \; ({\sf int},r1),\, rL)$$, and
$$({\sf ref} \; ({\sf int},rL),\, rL)$$, respectively. However, since
$$r1$$ outlives $$rL$$, we can define following subtype relation among
the types of both branches:<br />
$$({\sf ref} \; ({\sf int},r1),\, rL)$$ <: $$({\sf ref} \; ({\sf int},rL),\, rL)$$ <br />
The subtype relation effectively makes it safe to use a reference into
$$r1$$, wherever a reference into $$rL$$ is expected. Safety follows
from the observation that $$r1$$ outlives $$rL$$, hence if its safe to
dereference a pointer into $$rL$$, it should be safe to dereference a
pointer into $$r1$$. Further, the subtyping relation lets us unify the
types on both the branches to $$({\sf ref} \; ({\sf int},rL),\, rL)$$.
Hence $$x$$ must be considered a reference into region $$rL$$, or any
other region that lives atleast as long as $$rL$$.


### Formalization ###

Full static and dynamic semantics for Cyclone language are given in a
tech report. The formalization is of special interest to us, as the
language includes full imperative features, such as pointers and
aliasing. Small step operational semantics are presented as rewriting
relation from machine states to machine states, and gets stuck if a
memory unsafe operation is attempted. They prove the standard type
safety theorem - well-typed Cyclone programs don't get stuck.

### Comments ###

Outlives relation, and subtyping among reference types is the key
takeaway from this paper. Cyclone's regions are all statically nested
and stick to LIFO discipline. Therefore, transferable regions cannot
be modeled in Cyclone. It is certainly conceivable to extend the
formal language of Cyclone with constructs to allocate, open, close
and transfer ownership of transferable regions. However, Broom
is based on C#, which, unlike C, has no explcit pointers, and is
inherently memory safe. Therefore, Cyclone's formal model extended
with transferable regions could be an overkill, in the sense that it
allows such unsafe memory interactions that are not possible in Broom
programs. More work is needed to prove or refute this intuition.
Further, there are no higher-order functions in Cyclone's
formalization. I am not sure how often higher-order features are used
in Broom (and C#) programs, but if they are indispensable then
Cyclone's formal model may not be a good place to start.

Henglein, Makholm, & Niss (HMN01)
================================

HMN01 is a variant of T&T memory management and type system. Following
are salient points:

+ First-Order Programs. No higher-order functions. However, C style
  function pointers can be supported. Basically, closures are not
  allowed, thereby preventing _hidden_ effects that we discussed in
  T&T. The way this is manifest in the language is:
  - Absence of lambda expressions.
  - Presence of _declaration_ syntactic class ($$d$$), which declares
    a function. At the top level, program is $${\sf let} \; \bar{d} \;
    {\sf in} \; e$$
+ No hierarchic $${\sf letregion}$$ construct. Instead, embed an
  imperative region sublanguage in base language. The region
  sublanguage has constructs to allocate, release and alias regions.
  Allocation and release of of regions are completely independent.
  In other words, he lifetime of a region need not be delineated by a
  nice lexical block. Anything can be deallocated at any time.
  Therefore, the type system must be capable of identifying references
  that suddenly become dangling after a function call.

To compare with our problem setting:

+ We need higher-order state. Although explicit use of higher-order
  features in Broom/C# programs is minimal, there are classes, which
  encapsulate state that is implicitly shared among all methods of the
  class.
+ We need the $${\sf letregion}$$ construct to model statically nested
  regions in Broom.
+ The imperative region sublanguage supports transferable regions of
  Broom. The lifetime of a transferable region is split into multiple
  incarnations, with each incarnation nicely delineated by a lexical
  block. This feature should make our reasoning with dynamic regions
  easier than in HMN01.

### Base Types ###

Formal language includes boxed integer types, which are T&T style
decorated types: $$({\sf int},\, \rho)$$. The type represents an
address in region ρ which contains an integer. Since ρ is a value in
the language, the decorated types are dependent types (the paper
acknowledges this elephant in the room). Now, consider the following
example:

    [[new r1]]
    let x = 1 at r1
    in let y = [[new r2]] x + (1 at r2) at r2 [[release r1]]
       in e

where $$e$$ is some expression. The type of $$x$$ on line 2 is 
$$({\sf int},\, r1)$$. Now, what is its type in $$e$$? Since r1 has
been released, <!-- (and the region deallocated, as there are no other
aliases) --> the type of $$x$$ should now be _updated_ to 
$$({\sf int},\, \top)$$. The $$\top$$ in the type denotes that $$x$$
is an address into a released <!-- may not yet be deallocated --> 
region. This ability to track state changes in types makes the type
system a typestate system. Also, the type system can possibly ascribe
different types to a variable at different program locations; which
means that it is flow sensitive. 

### Typing Judgement ###

### Function Types ###

Functions in HMN01 have three kinds of region parameters - input
regions, constant regions, and output regions. The set of constant
regions is disjoint with the set of input and output regions. A
function is expected to read its input from input regions, and write
its output to output regions. Input and output regions need not be
disjoint, so same formal parameter can occur as an input region and
output region. Caller provides input and constant region arguments. As
soon as the call is made, the name to value bindings of actual region
arguments are lost in calling context. I understand this as a
pessimistic approach - Callee releases all its input regions, unless
it explicitly states otherwise.

## Important Questions to Answer ###

1. Since variables are already addresses into regions, do we need to
   include references in our formalization? If we consider the dynamic
   semantics, we definitely need to as we have to capture updatation
   of a memory location. But, for static semantics, we are only
   interesting in the state of a region, but not the content of
   memory locations within the region; so, does it matter?
2. HMN01 explicitly updates the types in Γ, when a ρ is released. My
   intuition was to use a separate environment $$\Gamma_R$$ for region
   handlers, and extend $$\Gamma_R$$ with new type of ρ. It would be
   interesting to understand how these methods compare, especially
   given that region handlers can be aliased and, unlike HMN01,
   releasing an alias deallocates the region in Broom.

Ownership Types for Real-Time Java (RTSJ03)
===========================================
Type system for writing real-time programs in Java. Guarantees that:

+ Deleting a region does not create dangling references,
+ Real-time threads do not access heap references

The type system is a unified framework of region types and ownership
types. Region types ensure that programs never follow dangling
references. Ownership types statically enforce object encapsulation
and enable modular reasoning about program correctness in
object-oriented programs.

If an object $$s$$ contains a reference to object $$v$$, how do we
know that $$v$$ is not referred outside $$s$$? Ownership types in
RTSJ03 allows the type of $$v$$ to declare that it is owned by $$s$$.
The type system ensures that there are no pointers to $$v$$ outside
$$s$$. Now, when $$s$$, and $$v$$ are allocated in same region $$r$$,
deallocating the region does not create any dangling references to
$$v$$.  We have similar requirement - deallocating a region should not
create dangling references. We also want a region to be
self-contained. Everything reachable from the root of a region should
be contained in the region. So, there is a reflexitive transitive
_ownership_ relation. However, ownership relation does not deal with
aliasing - it allows a reference to be aliased as long as the alias is
also stored in the same region; both aliases are still _owned_ by the
region. Aliasing is relevant in the context of region handlers being
first-class. RTSJ03 does not worry about where the region handlers are
stored, as region handlers are not first-class in RTSJ03. They are
introduced by lexical blocks, and follow LIFO discipline.

Ownership relation implies outlives relation in RTSJ03. This is
supposedly natural, since if $$o_1$$ owns $$o_2$$, then $$o_1$$
outlives $$o_2$$, as $$o_2$$ is accessible only from $$o_1$$. If
applied to regions, this seems to contradicts the the standard
semantics of outlives relation. As per convention, if region $$R_1$$
outlives region $$R_2$$, then $$R_1$$ is not allowed to contain
pointers into $$R_2$$, as they become dangling once $$R_2$$ is freed.
Maybe thats why ownership relation is not defined among regions in
RTSJ03. Region handlers are not first-class, so a region handler need
not be "stored" in other region. However, in our case region handlers
are first-class. Consider a case where handler for region $$R_2$$ is
stored in $$R_1$$. We can consider that $$R_1$$ _owns_ $$R_2$$, in the
same sense as RTSJ03:

1. No other region can contain a reference to region handler of
   $$R_2$$, and
2. $$R_1$$ should outlive $$R_2$$:
   - GC heap and immortal region can store all region handlers (i.e.,
     own all regions) $$\Rightarrow$$ they outlive all regions. 
   - If a static region stores a reference (i.e., owns) a transferable
     region, it better be the case that the region is transferred
     before static region's lifetime ends.
   - As per our current dynamic semantics (Calculus V3), a static
     region ($$R$$) owns itself by default as the region stores its
     own handler (E-LetReg rule). This is ok as a region outlives
     itself.  However, point (1) means that no other region (including
     the parent static region) can alias the region handler of $$R$$.

We conclude that _outlives_ and _ownership_ relations can be defined
over regions in Broom, with almost the same semantics as RTSJ03. One
change is that if $$R$$ stores region handlers of (owns) both $$R_1$$,
and $$R_2$$, even then $R_1$$ is not permitted to access/refer
$$R_2$$ (and vice versa).

We use _outlives_ relation in its actual sense - that region actually
outlives other region. This means that if $$R_1$$ outlives $$R_2$$,
then $$R_2$$ can contain pointers into $$R_1$$.

Therefore, our _ownership_ relation has lesser significance - 

+ Like ownership relation in RTSJ03, if $$R_1$$ _owns_ the handler of
  $$R_2$$, then no other region can contain reference to the handler
  of $$R_2$$. 
+ However, unlike RTSJ03, even $$R_1$$ cannot duplicate handlers of
  $$R_2$$. That is, $$R_1$$ should maintain the unique handler of
  $$R_2$$.
+ Further, $$R_1$$ need not outlive $$R_2$$. 

The Calculus
============
<div>
\[
\newcommand{\ALT}{~|~} 
\newcommand{\spec}{\forall \overline{a}. \phi
    \Rightarrow \psi} 
\newcommand{\cf}[1]{CoordFree(#1)}
\newcommand{\llbracket}{[\![} 
\newcommand{\rrbracket}{]\!]}
\newcommand{\conj}{\wedge}
\newcommand{\disj}{\vee}
\newcommand{\specs}{\overline{spec}}
\newcommand{\Conj}{\bigwedge}
\newcommand{\tuplee}[1]{\langle #1 \rangle}
\newcommand{\redsto}{\longrightarrow}
\newcommand{\spc}{\quad}
\]
</div>

We now take a first shot at a simple calculus that lets us model the
following aspects:

1. A combination of $${\sf letregion}$$ based static regions, and
   dynamic transferable regions, with $${\sf new}$$, $${\sf free}$$
   and $${\sf open}$$ constructs. While static region handlers live at
   the type level, handlers of transferable region are first class
   values of the language. Motivations behind this decision are:
   1. We do not want functions returning static region handlers.
      However, we want to allow functions that create transferable
      regions and return their handlers
   2. We want to reuse the existing T&T based infrastructure for
      static regions as much as possible.

   We do not have specially designated heap and stack regions.
   However:

   1. A heap region is a static region that is allocated at the
      beginning of the program, and deallocated at the end. If
      $$\rho_H$$ is the heap region handler, then the program is
      $${\sf letregion}\; \rho_H \; {\sf in} \; e$$, at the top level.
   2. A stack region (an activation record) is a static region whose
      lifetime is the lifetime of a function call. 
2. Closures, but not fixpoint.
3. The danling pointer problem resulting from accessing such variables
   in scope, which are addresses into a freed region.
4. The space leak problem, resulting from deallocating a static region
   storing the only reference to a transferable region.

Observations:

1. In T&T, it is possible that a function expects a region parameter
   even for local allocations. Although a good region inference
   algorithm prevents such cases, it is nevertheless sound.

## The calculus ##
$$
\begin{array}{lcl}
     n & \in & integers\\
  \rho & \in & static \; region \; handlers\\
  x    & \in & variables\\
  r    & ::= & e \ALT \rho\\
  e    & ::= & x \ALT e \; e \ALT \lambda x.\,e \; {\sf at} \; r \ALT n \; {\sf at} \; r
                \ALT e \otimes e \; {\sf at}\; r\ALT (e,e) \; {\sf at} \; r \ALT \pi_1 \, e \\
       &     & \ALT \pi_2 \, e \ALT {\sf new} \; {\sf at} r \ALT {\sf transfer}\; e \ALT {\sf open} \; e \; 
                {\sf in} \; e \; \ALT {\sf letregion} \; \rho \; {\sf in} \; e\\
\end{array}
$$

+ We have (boxed) integers in our language. We have pairs that serve
  as primitive models for composite datatypes. We should have a $${\sf
  unit}$$ type to assign the type to return value of $${\sf transfer}$$.
+ $$\pi_1$$ and $$\pi_2$$ are projection operations over pairs. In
  code, we use SML style `#1` and `#2`, instead.
+ The $${\sf new}$$ expression allocates a new region and returns a
  new region handler. Since region handler is a value, it has to be
  stored in some region. The $${\sf new} \; {\sf at} \; r$$ specifies
  that the region handler should be stored at $$r1$$. 

A well behaving example (Maybe not quite well behaving):

    letregion ρ1
    in let x1 = 1 at ρ1
       in let x2 = 2 at ρ1
          in  let r1 = new
                  in let y = open r1 
                             in let r1val = (x1,x2) at r1
                                in r1val
                     in transfer r1

An example of dereferencing an invalid address:

    letregion ρ1
    in let x1 = 1 at ρ1
       in let x2 = 2 at ρ1
          in  let r1 = new
                  in let y = open r1 
                             in let r1val = (x1,x2) at r1
                                in r1val
                     in let _ = transfer r1
                        in #1 y

Observe that y is an address into region r1, which has already been
transferred, when `#1 y` is executed.

An example of a space leak:

    let z = letregion ρ1
            in let x1 = 1 at ρ1
               in let x2 = 2 at ρ1
                  in  let r1 = new at ρ1
                          in let y = open r1 
                                     in let r1val = (x1,x2) at r1
                                        in r1val
                             in y
    in #2 z

Observe that in the above example, the only region handler for the
transferable region $$r1$$ is stored in static region ρ1, which is
deallocted before the region $$r1$$ is transferred. Now, although
$$z$$ is an address into region $$r1$$, and is also in scope for the
continuation, we have effectively lost the region handler for $$r1$$,
without which the region cannot be transferred.

## Calculus V2 ##

Rama's suggestions:

+ Static regions in Broom are more like dynamic regions in cyclone -
  region handlers are first class, and unlimited allocation is
  allowed. Moreover, we do not plan to extend the syntax of C#, so
  adding a new syntactic class to expression language is discouraged.
  Therefore, we should not separate ρ from $$x$$.
+ We need references eventually to model pointers in C#. If adding
  refernces does not complicate type checking, then we should include
  $${\sf ref}$$ in formalization. Gowtham's comments : I don't think
  adding references complicates type checking per se; but, experience
  of Standard ML designers suggests that it complicates type
  inference.  Since we are not yet concerned with inference, I think
  we can add references to the language.


$$
\begin{array}{lcl}
  n    & \in & integers\\
  x    & \in & variables\\
  \rho & \in & \mathit{region \; identifiers}\\
  e    & ::= & x \ALT e \; e \ALT \lambda x.\,e \; {\sf at} \; e \ALT n \; {\sf at} \; e
                \ALT e \otimes e \; {\sf at}\; e\ALT (e,e) \; {\sf at} \; e \ALT \pi_1 \, e \\
       &     & \ALT \pi_2 \, e \ALT {\sf new\tuplee{\rho}} \; {\sf at}\; e \ALT {\sf transfer}\; e \ALT {\sf open} \; e \; 
                {\sf in} \; e \; \ALT {\sf letregion}\tuplee{\rho} \; x \; {\sf in} \; e \ALT {\sf ref}\; e\\
  v    & ::= & x \ALT {\sf region}\tuplee{\rho} \ALT {\sf ref} \; v \ALT ...
\end{array}
$$

Kapil's summary of constraints on region system, with some added
comments:

1. "Regions are typed. Each region has a root object that can be
   accessed using a special property of the region object."
   - My guess is that primary motivation for this is that ease of
     garbage collection. A collector can collect all such objects in
     the region that are unreachable from the root.
   - This constraint is also sensible at the high-level, as a
     transferable region is semantically a message, therefore expected
     to have a well-defined structure.
2. "Objects in (immortal region)/(GC heap) can store references to other
   regions, but cannot reference objects inside other regions. This
   simplifies the type system, and will hold us in a good stead when we
   GC heap memory."
   - Let us consider a region type system with Tofte and Talpin style
     _decorated types_ (eg: $$({\sf int},\rho)$$), such that type of
     an object identifies the region (ρ) to which it belongs. In case
     of reference types, the type identifies the region to which
     referred object belongs, along with the region where the
     reference itself is stored (eg: $$({\sf ref} \; ({\sf
     int},\rho_1),\, \rho_2)$$). Assume there are no recursive
     datatypes (and fixpoint operator) in the language. Under this
     setting, if heap region contains references to objects within
     other regions, the type of such reference clearly identifies the
     region (through its lexically manifest identifier ρ) to which it
     is referring. If this region is freed, the type system can update
     the type(state) of references to objects in this region to convey
     that the pointers are now dangling. Clearly, there is no need to
     restrict heap from storing pointers to objects in other regions
     under this language setting.  Now, let us add polymorphic {\sf
     list } recursive datatype to the language. Since heap can now
     store lists, assume that heap stores a list of references to
     objects in other regions. However, since type system requires
     that all references in the list have same (decorated) type, all
     references should refer to objects of same type in same region.
     For example, such a list would have following type: $$(({\sf ref}
     \; ({\sf int},\rho_1),\, \rho_2) {\sf list}, \rho_H)$$, which
     indicates that spine of the list is stored in heap region
     ($$\rho_H$$), and list contains references to integers in region
     $$\rho_1$$, where references themselves are stored in region
     $$\rho_2$$. Now, if either of $$\rho_2$$ or $$\rho_1$$ are freed,
     the type system can update the type(state) of the list stored in
     heap, and can subsequently flag either accessing a reference in
     the list, or dereferencing such reference (resp.) as a type
     error. Therefore, it seems that even with a language containing
     recursive datatypes (or a fixpoint operator), there is no need to
     constrain heap from having pointers to objects in other regions.
     However, this conclusion is wrong as, with the extended language,
     region identifiers (ρ) need not be lexically manifest. For
     instance, consider a program that (1). creates several
     transferable regions, (2).  stores region handlers in a list
     $$l1$$, (3). iterates through the list $$l1$$ while flipping a
     coin, and when the coin lands heads for the first time, (4).
     stores several objects in the region at current position of the
     list, and stores references to such objects as a list ($$l2$$) in
     the heap region, and (5). exits.  Now, what is the type of the
     list $$l2$$? List $$l2$$ contains references to objects stored in
     $$\emph{some}$$ region, whose region handler is contained in
     $$l1$$, but not lexically manifest in the program. We can assign
     an existentially quantified type to $$l2$$, but it is very
     imprecise - if list $l1$ is unfolded again, and some region
     (handler) in the list is freed/transferred, we cannot be sure if
     references in list $$l2$$ refer to objects in freed region or
     not.
     A straightforward way to circumvent this problem is to avoid
     inter-region pointers, which is achieved through restrictions in
     (2) and (4). However, region handlers (or pointers to the roots
     of regions) have to be maintained somewhere, lest it leads to
     space leak. Therefore, let us relax the restriction, and allow
     region handlers to be stored in other regions. But, it leads to
     the same problem as before. To see how, consider a case when we
     have two lists containing region handlers to same set of regions,
     but in different order. We unfold one list and  free one of the
     regions. Now, which region handler in 2nd list is dangling?
     Observing that root of this problem is aliasing, we forbid
     aliasing of region handlers, which is achieved through
     restriction in (3).
3. "References to regions are linear, i.e., at any point, there can
   only be one reference to a region."
  - This solves the aliasing problem.
4. Objects in transferable regions cannot store references to other
   regions. This is to ensure that we can collect a region
   independently without having to worry about regions that may be
   referenced by objects in freed regions.
5. There may be references to objects within a region from stack
   variables. We shall provide a mechanism to identify lexical blocks
   where such references exist. The type system will ensure that
   regions cannot be freed from within a code region where a reference
   to an object in that region may exist.

Observations:

+ In Broom, transferable regions are similar to static regions, except
  that they (i) have split lifetimes, and (ii) can be transferred.
+ The ρ should be thought of as unique identifier for a region.
  Constructs $${new}\tuplee{\rho_1}$$ and $${letregion}\tuplee{\rho_2}$$
  introduce new transferable and static regions with identifiers
  $$\rho_1$$ and $$\rho_2$$, respectively. In actual implementation,
  compiler can introduce the ρ annotation.
+ The runtime value of a handler of region with id ρ is $${\sf
  region}\tuplee{\rho}$$.
+ All region handlers are first class values; so, wherever we expect
  to see a region handler, we see an expression ($$e$$) (that is
  expected to evaluate to a region handler) instead.

Questions:

+ How do we deal with region polymorphic functions, given that regions
  are now fully in expression language, instead of type language?
  Thoughts on prenex parameterization of region variables, and
  parameter instantiation inference?


<!-- 
Dynamic Semantics
=================

  Dont think too much about portability to C#. Modeling transferable
  regions in the context of T&T itself is an exercise in itself.
  And this exercise has been solved by WW01 :-)
-->

WW01 Examples
=============

    let gen = λ().-> alloc () in
    unpack ρ1, r1 = alloc () in
    unpack ρ2, r2 = alloc () in
    let! (r1) res = 
      let! (r2) res2 =
        let x = 1 at r1 in
        let y = 2 at r2 in
        let res = (y,x) at r2 in
          res in
      let (y,x) = res2 in 
      let res1 = (x,y) at r1 in
        (res1,res2) at r1
    let l1 = Cons (r1, Cons (r2, []))


